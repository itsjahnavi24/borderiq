# BorderIQ Architecture

| Field | Value |
|---|---|
| Title | BorderIQ Architecture Views |
| Version | 1.0 |
| Status | Draft for review |
| Owner | Jahnavi Ralhan |
| Last updated | 2026-08-01 |
| Scope | Rendered architecture views for the current MVP and the target platform. Each diagram carries a short explanation and a status label. Component contracts live in the [LLD](LLD.md); requirements in the [PRD](PRD.md). |
| Related documents | [PRD](PRD.md), [LLD](LLD.md), [API_SPEC](API_SPEC.md), [DATA_MODEL](DATA_MODEL.md), [DECISION_ENGINE](DECISION_ENGINE.md), [TOOL_CALLING](TOOL_CALLING.md), [KNOWLEDGE_GRAPH](KNOWLEDGE_GRAPH.md), [SECURITY](SECURITY.md), [DEPLOYMENT](DEPLOYMENT.md), [ROADMAP](ROADMAP.md), [AGENTS](AGENTS.md) |

Labelling follows the convention defined in the [PRD](PRD.md). Diagrams 1 and 2 show what exists today (with the backend portions marked as described, pending verification under OQ-01). Diagrams 3 onward are Target architecture (Proposed) unless stated otherwise. Component identifiers (C-01 to C-23) match the LLD Section 6 inventory.

## 1. Current MVP Context Diagram

The MVP is a single-user, single-tenant demonstration: a browser client, a Flask backend over a local vector store, and one external AI provider. There is no durable enterprise data.

```mermaid
flowchart LR
    U["Compliance user"] --> FE["Web frontend SPA<br/>Current MVP verified"]
    FE -->|"JSON over HTTP"| BE["Flask backend<br/>Current MVP described"]
    BE --> VDB[("ChromaDB<br/>local persistence")]
    BE -->|"embeddings and chat"| OAI["OpenAI APIs<br/>external"]
    PDF["Official CBAM PDFs<br/>3 documents"] -->|"one-time ingestion"| BE
```

## 2. Current MVP Container Diagram

The frontend carries a demo rules engine and demo datasets, all labelled in the UI; the backend carries retrieval and narration. Nothing writes durable state.

```mermaid
flowchart TB
    subgraph Browser["Browser - Current MVP verified"]
        SPA["SPA: 8 surfaces<br/>dashboard, assistant, analyzer,<br/>knowledge, portfolio, updates,<br/>audit trail, settings"]
        DRE["Client demo rules engine<br/>Representative demo data"]
        SAL["Session audit log<br/>in-memory, volatile"]
        SPA --> DRE
        SPA --> SAL
    end
    subgraph Server["Flask app - Current MVP described"]
        ASK["POST /api/ask"]
        ST["GET /api/status"]
        KB["GET /api/knowledge-base"]
        RAG["RAG service<br/>load, chunk, embed, retrieve, narrate"]
        CAT["Product catalogue<br/>described"]
        ASK --> RAG
    end
    SPA --> ASK
    SPA --> ST
    SPA --> KB
    RAG --> VDB[("ChromaDB local")]
    RAG --> OAI["OpenAI embeddings and chat"]
```

## 3. Target Enterprise Context Diagram

Target architecture (Proposed). BorderIQ sits between enterprise operational systems and the compliance obligation, consuming authoritative regulation on one side and enterprise records on the other, and emitting reviewed, replayable decisions.

```mermaid
flowchart LR
    subgraph Users["People"]
        CA["Compliance analyst"]
        RV["Reviewer"]
        AU["Auditor"]
        CO["Consultant multi-client"]
    end
    subgraph Ext["External systems"]
        ERP["ERP and plant systems"]
        VER["Verifier systems"]
        CP["Carbon price service<br/>pending OQ-07"]
        EU["Authoritative EU sources"]
    end
    BIQ["BorderIQ platform<br/>compliance investigations,<br/>policy evaluation, review,<br/>audit packages"]
    CA --> BIQ
    RV --> BIQ
    AU --> BIQ
    CO --> BIQ
    ERP -->|"shipments, products, emissions records"| BIQ
    VER -->|"verification reports"| BIQ
    CP -->|"price references"| BIQ
    EU -->|"regulation versions"| BIQ
    BIQ --> LLM["Model provider<br/>governed, versioned"]
```

## 4. Target Container Diagram

Target architecture (Proposed). One modular monolith (LLD Section 5) with strict module seams, plus external stores. Arrows are in-process calls until the extraction thresholds in LLD Section 36.

```mermaid
flowchart TB
    FE["Web frontend C-01"] --> API["API layer C-02<br/>auth, validation, rate limits"]
    subgraph App["Modular monolith"]
        API --> ORCH["Investigation orchestrator C-03"]
        API --> REV["Review service C-19"]
        API --> KSVC["Knowledge service"]
        ORCH --> TREG["Tool registry and executor C-04"]
        TREG --> CALC["Calculation engine C-05"]
        TREG --> POL["Policy engine C-06"]
        TREG --> RISK["Risk engine C-07"]
        TREG --> SCEN["Scenario engine C-08"]
        TREG --> RET["Retrieval service C-10"]
        TREG --> ERS["Entity resolution C-12"]
        ORCH --> ASM["Decision assembler C-09"]
        ING["Ingestion workers C-11"]
        CHG["Change intelligence C-23"]
    end
    ASM --> TRC[("Audit and trace store C-20")]
    App --> PG[("PostgreSQL C-13")]
    App --> OBJ[("Object storage C-14")]
    RET --> VIX[("Vector and lexical index C-15")]
    App --> Q[("Queue and outbox C-17")]
    App --> CCH[("Cache C-18")]
    App -.->|"optional, fail closed"| KG[("Knowledge graph C-16")]
    API --> IDP["Identity provider C-21"]
    App --> OBS["Observability C-22"]
    App --> SLLM["Model provider"]
```

## 5. Shipment-Investigation Sequence

Target architecture (Proposed, Phase 2 onward). The orchestrator drives the 16-stage lifecycle from [DECISION_ENGINE.md](DECISION_ENGINE.md); this view compresses it to the principal interactions. Figures come only from the calculation engine; the model narrates after assembly.

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API layer
    participant O as Orchestrator
    participant T as Tools and engines
    participant R as Retrieval
    participant D as Decision assembler
    participant S as Trace store
    C->>A: POST investigation (shipment)
    A->>O: validated request, correlation ID
    O->>T: resolve product and CN code
    O->>T: check evidence sufficiency
    O->>T: run calculations (default and actual)
    O->>T: evaluate policy eligibility
    O->>T: classify risk
    O->>R: retrieve supporting passages (scoped)
    R-->>O: cited fragments or abstention
    O->>D: assemble structured decision
    D->>S: persist decision and full trace
    alt review required
        D-->>C: status review_required with task
    else auto approved
        D-->>C: finalised decision with citations
    end
```

## 6. Regulatory-Question Sequence

Current MVP (described) implements the unscoped form of this flow; the scoped pipeline with sufficiency checks and abstention is Target architecture (Proposed, Phase 5).

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API layer
    participant R as Retrieval service
    participant L as LLM narration
    participant S as Trace store
    C->>A: POST question
    A->>R: scoped query (jurisdiction, effective date)
    R->>R: metadata filter, dense plus lexical, rerank
    alt sufficient evidence
        R-->>A: ranked cited fragments
        A->>L: narrate from fragments only
        L-->>A: draft answer
        A->>A: citation validation, numeric guard
        A->>S: trace query, fragments, versions
        A-->>C: answer with citations
    else insufficient evidence
        R-->>A: sufficiency below threshold
        A->>S: trace abstention
        A-->>C: explicit abstention response
    end
```

## 7. Tool-Calling Sequence

Target architecture (Proposed, Phase 2). The model selects tools from the registry; the executor owns validation, permissions, execution, and tracing. The model never receives unvalidated output and cannot fabricate a result ([TOOL_CALLING.md](TOOL_CALLING.md)).

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant L as LLM planner
    participant G as Tool registry
    participant X as Tool executor
    participant T as Tool service
    participant S as Trace store
    O->>L: context plus allowed tool list
    L-->>O: tool selection with arguments
    O->>G: lookup tool version and schema
    G-->>O: definition, permission, approval class
    alt input invalid or not permitted
        O->>S: trace rejection
        O-->>L: typed rejection (no execution)
    else valid and permitted
        O->>X: execute with timeout and idempotency key
        X->>T: typed call
        T-->>X: raw result
        X->>X: validate output schema
        X->>S: trace inputs, outputs, duration
        X-->>O: validated result
        O-->>L: validated result only
    end
```

## 8. Regulatory Ingestion Pipeline

Target architecture (Proposed, Phase 5). Current MVP (described) implements the minimal load-chunk-embed path without registry, checksums, or structural parsing. Idempotent by checksum; changed files create versions, never overwrites.

```mermaid
flowchart LR
    SRC["Authoritative source<br/>allowlist only"] --> FETCH["Acquire file"]
    FETCH --> RAW["Immutable raw store<br/>checksum plus provenance"]
    RAW --> REG["Document registry<br/>version, effective dates,<br/>supersession links"]
    REG --> PARSE["Structural parsing<br/>article, annex, table, section<br/>page fallback"]
    PARSE --> FRAG["Fragments plus metadata<br/>CN codes, methodology,<br/>emissions type, window"]
    FRAG --> EMB["Embedding"]
    EMB --> VIX[("Vector index")]
    FRAG --> LEX[("Lexical index")]
    VIX --> VER["Verification pass<br/>counts and spot citations"]
    LEX --> VER
```

## 9. Regulatory-Change Workflow

Target architecture (Proposed, Phase 5). No policy changes without recorded human approval (AIR-010); the change itself becomes part of the audit record.

```mermaid
flowchart TD
    NEW["New regulation version ingested"] --> DIFF["Structural diff vs superseded version"]
    DIFF --> CLS["Classify changed articles,<br/>annexes, tables"]
    CLS --> MAP["Impact mapping:<br/>CN codes, routes, methodologies,<br/>active policies, open decisions"]
    MAP --> TASK["Create human review task<br/>with impact report"]
    TASK --> HUM{"Qualified reviewer decision"}
    HUM -->|"approve with effective date"| ACT["Activate policy version"]
    HUM -->|"reject"| REC["Record rejection with rationale"]
    ACT --> RERUN["Re-run affected open investigations"]
    ACT --> FLAG["Flag affected finalised decisions"]
```

## 10. Human-Review Workflow

Target architecture (Proposed, Phase 4). Routing rules are listed in PRD Section 25; every reviewer action joins the decision trace.

```mermaid
flowchart TD
    DEC["Decision assembled"] --> RQ{"Routing rules match?"}
    RQ -->|"no"| AUTO["Status: auto approved"]
    RQ -->|"yes"| TASK["Review task created<br/>full context and evidence"]
    TASK --> RVW{"Reviewer action"}
    RVW -->|"approve"| FIN["Status: finalised<br/>action recorded"]
    RVW -->|"reject"| BLK["Status: blocked<br/>rationale recorded"]
    RVW -->|"request information"| INFO["Status: insufficient evidence<br/>required actions listed"]
    INFO --> RESUB["Evidence added,<br/>investigation re-runs"]
    RESUB --> DEC
```

## 11. Decision-Replay Workflow

Target architecture (Proposed, Phase 2 trace, Phase 4 full package). Replay reads only the stored trace and pinned versions; it never touches live systems, and must reproduce figures byte-identically (NFR-001).

```mermaid
flowchart LR
    AUD["Auditor request<br/>decision ID"] --> LOAD["Load trace:<br/>input snapshot, document versions,<br/>policy, engine, model, prompt versions"]
    LOAD --> REX["Re-execute calculation and policy<br/>with pinned versions"]
    REX --> DIFFR{"Byte-identical to stored figures?"}
    DIFFR -->|"yes"| PKG["Emit audit package"]
    DIFFR -->|"no"| ALERT["Integrity alert:<br/>replay divergence investigation"]
```

## 12. Batch-Ingestion Workflow

Target architecture (Proposed, Phase 3). Per-row outcomes, never all-or-nothing; ambiguous mappings route to review instead of guessing (FR-010, FR-011).

```mermaid
flowchart TD
    UP["CSV or Excel upload<br/>or batch API"] --> FVAL["File-level validation<br/>schema and idempotency key"]
    FVAL --> JOB["Import job created"]
    JOB --> ROWS["Per-row: validate,<br/>canonicalise, resolve entities"]
    ROWS --> OK["Accepted rows persisted"]
    ROWS --> AMB["Ambiguous mappings<br/>to review queue"]
    ROWS --> REJ["Rejected rows<br/>with actionable reasons"]
    OK --> INV["Investigations queued"]
    OK --> RPT["Import report"]
    AMB --> RPT
    REJ --> RPT
```

## 13. Event-Driven Workflow

Target architecture (Proposed, Phase 3): transactional outbox with a lightweight queue; idempotent consumers; dead-letter after bounded retries. Kafka-class platforms only past the thresholds in [DEPLOYMENT.md](DEPLOYMENT.md).

```mermaid
flowchart LR
    SVC["Service transaction"] -->|"same transaction"| DB[("State tables")]
    SVC -->|"same transaction"| OBX[("Outbox table")]
    OBX --> DISP["Dispatcher"]
    DISP --> Q[("Queue<br/>schema-versioned envelopes,<br/>correlation IDs")]
    Q --> CON["Consumer<br/>idempotent by event ID"]
    CON -->|"retry with backoff"| Q
    CON -->|"bounded attempts exceeded"| DLQ[("Dead-letter queue")]
    OBX -.->|"replay source"| DISP
```

## 14. Multi-Tenant Isolation View

Target architecture (Proposed, Phase 9). Tenancy is enforced at every layer by construction; the repository layer refuses tenant-less access, and NFR-007 tests verify zero cross-tenant results.

```mermaid
flowchart TD
    REQ["Request with identity"] --> AUTHN["Authenticate<br/>resolve tenant and role"]
    AUTHN --> CTX["Tenant context bound<br/>to correlation ID"]
    CTX --> REPO["Repository layer<br/>mandatory tenant_id filter"]
    REPO --> PGT[("PostgreSQL rows<br/>tenant_id scoped")]
    CTX --> RETT["Retrieval<br/>tenant-scoped namespace"]
    RETT --> VNS[("Vector namespaces<br/>per tenant")]
    CTX --> OBJT[("Object storage<br/>tenant prefixes")]
    CTX --> KGT[("Graph isolation<br/>per KNOWLEDGE_GRAPH.md")]
    REPO -.->|"cross-tenant query"| DENY["Fail closed: denied and logged"]
```

## 15. Deployment View

Target architecture (Proposed); the authoritative environment matrix and thresholds live in [DEPLOYMENT.md](DEPLOYMENT.md). One application image serves API and workers; managed stores; no Kubernetes before its documented threshold.

```mermaid
flowchart TB
    U["Users"] --> LB["Load balancer / TLS"]
    LB --> APPC["App containers<br/>API plus services"]
    APPC --> WRK["Worker containers<br/>same image: investigations,<br/>imports, ingestion"]
    APPC --> PG[("Managed PostgreSQL")]
    APPC --> OBJ[("Object storage")]
    APPC --> VIX[("Vector index<br/>per OQ-03")]
    APPC --> CCH[("Cache")]
    APPC --> Q[("Queue")]
    APPC --> SEC["Secrets manager"]
    APPC --> OBS["Observability:<br/>logs, metrics, traces"]
    CI["CI/CD pipeline<br/>tests, evals, migrations"] --> APPC
```

## 16. Trust Boundaries

Target architecture (Proposed); detailed threat model in [SECURITY.md](SECURITY.md). Retrieved regulation text and uploaded documents are data inside the application zone, never instructions (AIR-006).

```mermaid
flowchart LR
    subgraph Z1["Zone 0: untrusted"]
        BR["Browser client"]
        UPL["Uploaded files"]
    end
    subgraph Z2["Zone 1: application"]
        API["API layer<br/>authn, validation"]
        ORC["Orchestrator and engines"]
        SAN["Upload scanning<br/>and type validation"]
    end
    subgraph Z3["Zone 2: data"]
        PG[("PostgreSQL")]
        OBJ[("Object store")]
        VIX[("Indexes")]
    end
    subgraph Z4["Zone 3: external providers"]
        LLM["Model provider"]
        EUS["EU sources allowlist"]
    end
    BR -->|"TLS, authenticated"| API
    UPL --> SAN --> OBJ
    API --> ORC
    ORC --> PG
    ORC --> VIX
    ORC -->|"minimum necessary context"| LLM
    EUS -->|"checksummed ingestion"| OBJ
    VIX -->|"retrieved text as untrusted data"| ORC
```

## 17. Data-Lineage Diagram

Target architecture (Proposed, Phases 2 to 5). Every figure and claim in a decision traces back to versioned sources; the audit package is the closure of this graph for one decision (FR-020).

```mermaid
flowchart LR
    EUDOC["Source document version<br/>checksum, effective window"] --> FRAG["Regulatory fragment<br/>article or table, page span"]
    FRAG --> RTRACE["Retrieval trace<br/>scores, filters, versions"]
    SHIP["Shipment input snapshot"] --> RESOL["Entity resolution record<br/>method, confidence, provenance"]
    RESOL --> CRUN["Calculation run<br/>formula version, units, factors"]
    FACT["Emission factor<br/>source, validity window"] --> CRUN
    POLV["Policy version<br/>approval record"] --> PEVAL["Policy evaluation"]
    CRUN --> DECN["Structured decision"]
    PEVAL --> DECN
    RTRACE --> DECN
    RVA["Reviewer actions"] --> DECN
    DECN --> APKG["Audit package<br/>exportable closure"]
```
