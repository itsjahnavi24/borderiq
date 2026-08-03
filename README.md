# BorderIQ

Enterprise CBAM compliance intelligence and decision-orchestration platform.

BorderIQ helps manufacturers, exporters, and their advisors answer one operational question per shipment: can this shipment be reported using the proposed emissions methodology, what evidence supports that decision, what is the estimated financial impact, and what must happen next? It combines retrieval over official EU CBAM documents with a deterministic calculation layer, and is designed to grow into a governed investigation platform with policy evaluation, human review, and replayable audit packages.

Architectural principle, applied everywhere: the language model interprets, plans, classifies, extracts, and narrates. It never invents or independently calculates authoritative compliance figures. Deterministic services calculate. Policy engines evaluate. Retrieval provides cited evidence. Humans approve high-impact decisions.

## Project Status

Early-stage. This repository contains a working local MVP plus a complete design suite for the target platform. The two are strictly separated everywhere in this repository using the labelling convention defined at the top of [docs/PRD.md](docs/PRD.md):

- **Current MVP (verified)**: working code in this repository, runs locally.
- **Target architecture (Proposed)**: designed in `docs/`, not built. Nothing in the documentation should be read as implemented unless labelled as such.

The MVP is local-development software: it has no authentication and is not internet-exposable as-is. See [docs/SECURITY.md](docs/SECURITY.md) Section 17 for the exact gap list and the phase each gap closes in.

## Current MVP (Verified)

What works today, end to end:

- **Regulatory Q&A with citations**: questions are answered only from retrieved passages of the indexed official CBAM documents; every answer carries source file, page number, document type, and chunk ID.
- **Shipment analysis**: for eight CBAM products (iron and steel, aluminium, cement, fertilisers), the backend resolves the official default emissions value (verified against the source PDF, with page provenance), runs a deterministic actual-versus-default calculation, and returns status, risk band, avoided emissions, estimated cost difference, a recommendation, and the supporting regulation reference. The language model plays no part in these figures.
- **Full-corpus retrieval index**: all pages of the three official documents (roughly 2,500 pages) are chunked (recursive, 1000 characters with 200 overlap, page metadata preserved), embedded, and stored in a local ChromaDB index with stable chunk IDs.
- **Live system status**: document, page, and chunk counts are computed from the real index and ingestion manifest, not hardcoded.
- **Honest degradation**: without an API key or index, the app still starts, reports its degraded state, and the frontend switches to a clearly labelled demo mode.
- **Web frontend**: an eight-section dashboard application (executive dashboard, compliance assistant, shipment analyzer, knowledge centre, portfolio analytics, regulatory updates, session audit trail, settings). Portfolio analytics and the regulatory timeline are labelled representative demo content.
- **Tests**: a pytest suite covers the deterministic calculator, including the reference case pinned end to end (500 t pig iron at 1.80 versus the 2.07 default at EUR 80 yields 135 tCO2e avoided and EUR 10,800).

## Quick Start

Prerequisites: Python 3.10+, an OpenAI API key, and the three official CBAM PDFs in `data/cbam_docs/`.

```bash
pip install -r requirements.txt

cp .env.example .env        # then put your real key in .env (gitignored)

python -m rag.rag_pipeline  # one-time index build, all pages (a few minutes)

python app.py               # http://127.0.0.1:5000
```

Verify: `python -m pytest rules_engine/` (no network needed), then open the dashboard and confirm live counts.

### Environment Variables

| Variable | Required | Purpose |
|---|---|---|
| `OPENAI_API_KEY` | For AI routes | Embeddings and chat. Without it the app starts, `/api/status` reports the degraded state, and the AI routes return 503 with guidance |

Model identifiers are currently constants in code (`gpt-4.1`, `text-embedding-3-small`). Never commit `.env`; if a key is ever exposed, rotate it immediately.

## API (Current)

| Route | Purpose |
|---|---|
| `POST /api/ask` | Citation-backed regulatory question answering |
| `POST /api/analyze-shipment` | Deterministic shipment analysis with supporting regulation retrieval |
| `GET /api/products` | The verified product catalogue (single source of truth for the frontend) |
| `GET /api/status` | Live index and service status |
| `GET /api/knowledge-base` | Indexed document manifest |

Exact request and response shapes, plus the designed target v1 API: [docs/API_SPEC.md](docs/API_SPEC.md).

## Repository Structure

```
app.py                    Flask entry point and API routes
rag/                      Retrieval pipeline: loader, splitter, vector store,
                          retriever, prompt builder, LLM client, ingestion
rules_engine/             Deterministic layer: product catalogue (source-verified),
                          calculator, shipment and dashboard services, tests
templates/, static/       Web frontend (Bootstrap 5, vanilla JavaScript)
data/cbam_docs/           Official CBAM PDFs (gitignored; supply your own copies)
docs/                     The design suite (see Documentation)
chroma_db/, knowledge_base.json   Generated by ingestion (gitignored)
```

## Tech Stack (Current)

Python, Flask, LangChain (PyPDFLoader, RecursiveCharacterTextSplitter, Chroma integration), ChromaDB (local persistence), OpenAI (`gpt-4.1` via the Responses API, `text-embedding-3-small`), Bootstrap 5, vanilla JavaScript, pytest. Pinned in `requirements.txt`.

## Target Platform (Proposed, Not Built)

The design suite specifies the evolution into a compliance decision platform: structured shipment investigations through a 16-stage lifecycle; a versioned, typed tool registry where the model selects tools but cannot fabricate outputs; deterministic calculation with decimal-safe units and golden tests; policy-as-code with effective dating and human-approved activation; hybrid effective-date-scoped retrieval with abstention; entity resolution with review queues; decision traces and byte-identical replay; audit packages; scenario analysis on decision twins; an optional fail-closed knowledge graph; enterprise integrations; and multi-tenant operation. Phasing with acceptance criteria: [docs/ROADMAP.md](docs/ROADMAP.md).

## Differentiation

BorderIQ does not compete with foundation models; it turns them into governed participants inside a specialised, deterministic, auditable compliance workflow. General-purpose models explain regulations. BorderIQ is designed to reconcile enterprise data with the applicable rule version, execute validated calculations, and produce a decision package that can be reviewed and replayed. Retrieval-augmented generation is one subsystem, not the product.

## Documentation

| Document | Contents |
|---|---|
| [docs/PRD.md](docs/PRD.md) | Product requirements: 50 requirements with acceptance criteria, personas, journeys, risks, open questions |
| [docs/LLD.md](docs/LLD.md) | Low-level design: 23 components, current and target architecture, migration plan |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | 17 rendered architecture views |
| [docs/DATA_MODEL.md](docs/DATA_MODEL.md) | 30 target entities, units, decimals, effective dating, retention |
| [docs/DECISION_ENGINE.md](docs/DECISION_ENGINE.md) | The 16-stage investigation lifecycle, decision object, state machine |
| [docs/TOOL_CALLING.md](docs/TOOL_CALLING.md) | Tool registry: 30 governed tools with schemas |
| [docs/API_SPEC.md](docs/API_SPEC.md) | Current routes (verified shapes) and the target v1 API |
| [docs/KNOWLEDGE_GRAPH.md](docs/KNOWLEDGE_GRAPH.md) | Optional graph projection: nodes, edges, queries, fail-closed contract |
| [docs/SECURITY.md](docs/SECURITY.md) | Threat model, OWASP LLM alignment, verified MVP gap list |
| [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) | Verified local run, target topology, environments, DR |
| [docs/ROADMAP.md](docs/ROADMAP.md) | Phases 0 to 10 with exit criteria |
| [docs/AGENTS.md](docs/AGENTS.md) | The three justified agents and why everything else is not one |

## Limitations (Current)

Local, single-user, single-tenant; no authentication; Flask development server; no durable business data beyond the index and manifest; product catalogue limited to eight goods; retrieval is unscoped dense top-k (effective-date scoping is a designed Phase 5 capability); the session audit trail is browser-memory only; frontend assets load from public CDNs; portfolio analytics and the regulatory timeline are representative demo data and labelled as such in the UI.

## Disclaimer

BorderIQ output is decision support, not legal advice, and the calculation engine is not a certified CBAM liability engine. Statements about CBAM in this repository are engineering context, not regulatory guidance. Synthetic and demo data is labelled wherever it appears.

## License

No license file has been added yet, so default copyright applies. Choosing and adding a `LICENSE` file is an open task; until then, this repository does not claim any open-source license.
