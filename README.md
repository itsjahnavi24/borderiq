# BorderIQ

> Enterprise AI Compliance Intelligence Platform for Carbon Border Adjustment Mechanism (CBAM)

BorderIQ is an AI-native compliance intelligence platform that helps enterprises interpret complex sustainability regulations, evaluate shipment-level compliance, and generate deterministic, evidence-backed compliance decisions.

Unlike traditional AI chatbots, BorderIQ combines Retrieval-Augmented Generation (RAG), deterministic policy engines, enterprise tool orchestration, and explainable AI to transform regulations into actionable compliance workflows.

---

## Why BorderIQ?

Modern environmental regulations such as the EU Carbon Border Adjustment Mechanism (CBAM) require organizations to:

- Interpret hundreds of pages of legal documentation
- Classify products correctly
- Calculate embedded carbon emissions
- Estimate financial exposure
- Validate supplier declarations
- Produce audit-ready compliance evidence

Today, much of this workflow is manual.

BorderIQ automates regulatory interpretation while ensuring every recommendation remains transparent, reproducible, and grounded in official regulations.

---

# Key Capabilities

### AI Compliance Assistant

Ask natural language questions about CBAM regulations and receive citation-backed responses grounded in official documents.

---

### Shipment Intelligence Engine

Analyze individual shipments by combining:

- Product classification
- Default emission factors
- Actual emissions
- Carbon pricing
- Compliance policies

to generate deterministic compliance assessments.

---

### Regulatory Intelligence

Retrieve official regulatory evidence using semantic search instead of relying solely on LLM memory.

---

### Policy Engine

Separates business logic from AI.

Every emissions calculation, compliance score, and financial estimate is executed using deterministic Python rules.

---

### Decision Engine

Transforms shipment data into structured compliance decisions instead of conversational responses.

---

### Explainable AI

Every recommendation includes:

- supporting regulations
- page references
- executed calculations
- reasoning trace

allowing compliance officers to audit every decision.

---

## Current MVP

- Executive Dashboard
- Compliance Assistant
- Shipment Analyzer
- Knowledge Centre
- Portfolio Analytics (demo)
- Audit Trail

---

# Enterprise Vision

BorderIQ is evolving beyond a regulatory chatbot into an Enterprise Compliance Operating System capable of:

- AI Tool Calling
- Multi-Agent Compliance Investigation
- Compliance Knowledge Graph
- Scenario Simulation
- Regulatory Change Detection
- Enterprise ERP Integration
- Policy-as-Code
- Human-in-the-Loop Approvals
- Audit Package Generation

---

# Technology Stack

## Frontend

- HTML5
- CSS3
- Bootstrap
- JavaScript

## Backend

- Python
- Flask

## AI

- OpenAI GPT
- Retrieval-Augmented Generation (RAG)
- LangChain

## Vector Search

- ChromaDB
- OpenAI Embeddings

## Business Logic

- Deterministic Python Rules Engine
- Compliance Policy Engine

## Data

- Official CBAM Regulations
- Product Catalogue
- Emission Factors
- Synthetic Enterprise Shipment Data

---

# High-Level Architecture

```text
                   User
                     │
                     ▼
           AI Compliance Planner
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   Retrieval     Policy Engine   Tool Calling
        │            │            │
        ▼            ▼            ▼
 ChromaDB      Rules Engine   Enterprise Data
        │            │            │
        └────────────┼────────────┘
                     ▼
             Decision Engine
                     │
                     ▼
          Explainable Recommendation
```

---

# Repository Structure

```text
docs/
│
├── PRD.md
├── LLD.md
├── ARCHITECTURE.md
├── AGENTS.md
├── TOOL_CALLING.md
├── API_SPEC.md
├── DATA_MODEL.md
├── KNOWLEDGE_GRAPH.md
├── DECISION_ENGINE.md
├── SECURITY.md
├── DEPLOYMENT.md
└── ROADMAP.md
```

---

# Documentation

| Document | Description |
|----------|-------------|
| PRD | Product Requirements Document |
| LLD | Low Level Design |
| Architecture | System Architecture |
| API Spec | API Contracts |
| Tool Calling | Enterprise Tool Registry |
| Decision Engine | Compliance Decision Workflow |
| Data Model | Core Business Entities |
| Knowledge Graph | Entity Relationship Model |
| Security | Enterprise Security Design |
| Deployment | Cloud Architecture |
| Roadmap | Future Product Vision |

---

# Future Roadmap

- AI Planning Agent
- Enterprise Tool Calling
- Regulatory Knowledge Graph
- Compliance Decision Engine
- Scenario Simulation
- Regulatory Change Detection
- Multi-Regulation Support
- ERP Integrations
- Multi-Tenant SaaS Platform

---

# License

MIT License
