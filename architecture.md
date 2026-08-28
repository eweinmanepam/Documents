# StableImpact — Architecture (Roadmap-aligned)

Source document: `stableimpact-roadmap-google-cloud-enterprise-en.md`

This document contains:

- Target architecture overview
- Requirements (functional, non-functional, prohibitions)
- Dataflow diagrams (sequence diagrams)
- Agent specification (roles, tool access policy, safety constraints, evaluation criteria)

---

## 1) Target Architecture Diagram

```mermaid
flowchart TB
  %% =========================
  %% StableImpact Target Architecture (Roadmap-aligned)
  %% =========================

  subgraph Channels["User channels"]
    GE["Gemini Enterprise<br/>(Operator/Reviewer/Analyst entry)"]
    PORTAL["External portal<br/>(Nonprofit + Donor)"]
    REVIEW["Reviewer console<br/>(Authenticated decision)"]
    DASH["Audit & impact dashboard<br/>(Looker/BigQuery views)"]
  end

  subgraph AgentPlatform["Gemini Enterprise Agent Platform"]
    RUNTIME["Agent Runtime<br/>(Managed execution)"]
    ORCH["ADK Orchestrator Agent"]
    SUBS["Specialist Agents<br/>(campaign/compliance/matching/evidence/treasury/audit/ops)"]
    SEARCH["Agent Platform Search<br/>(policies, guidance)"]
    SESS["Sessions<br/>(conversational continuity)"]
    GOV["Agent Identity / Registry / Gateway<br/>(when available)"]
    ARMOR["Model Armor<br/>(prompt/URL/PII inspection)"]
  end

  subgraph CloudRun["Cloud Run services"]
    MCP["StableImpact MCP<br/>(authoritative domain tools)"]
    API["Domain Backend API<br/>(state machine, ledger, authz)"]
    APPROVAL["Approval Service<br/>(record human decisions)"]
    LISTENER["EVM Event Listener<br/>(chain->Pub/Sub)"]
    SIGNER["Controlled Signer Adapter<br/>(prepares exact tx; no keys exposed to agents)"]
    INTEG["Enterprise Integrations Adapter<br/>(or Application Integration)"]
  end

  subgraph Data["Data & evidence"]
    SQL["Cloud SQL (PostgreSQL)<br/>(source of truth)"]
    GCSQ["Cloud Storage Quarantine<br/>(restricted)"]
    GCSS["Cloud Storage Sanitized<br/>(agent-safe)"]
    DLP["Sensitive Data Protection<br/>(redact before agent use)"]
    DOC["Document AI<br/>(extraction)"]
    BQ["BigQuery<br/>(audit, reconciliation, metrics, traces)"]
    SM["Secret Manager<br/>(RPC/OAuth/provider secrets refs)"]
  end

  subgraph Events["Event plane"]
    PS["Pub/Sub + Eventarc<br/>(normalized events)"]
    WF["Workflows<br/>(deterministic post-approval path)"]
  end

  subgraph Chain["Blockchain (Base Sepolia)"]
    RPC["Dedicated RPC (preferred)<br/>Public RPC fallback (demo only)"]
    CONTRACT["Escrow Smart Contract<br/>(test-USDC custody + milestone release)"]
    WALLET["Human execution wallet<br/>(separate from reviewer identity)"]
    DONOR["Donor wallet<br/>(test-USDC contribution)"]
  end

  %% ---- Flows ----
  GE --> RUNTIME --> ORCH --> SUBS
  ORCH --> SEARCH
  ORCH <--> SESS
  ARMOR --> ORCH
  GOV --> MCP

  SUBS --> MCP
  MCP --> API
  MCP --> INTEG

  PORTAL --> API
  REVIEW --> APPROVAL
  APPROVAL --> API

  API --> SQL
  API --> GCSQ
  GCSQ --> DLP --> GCSS
  GCSS --> DOC --> SQL

  API --> PS
  LISTENER --> PS
  PS --> WF
  APPROVAL --> WF
  WF --> SIGNER --> WALLET --> RPC --> CONTRACT
  DONOR --> RPC --> CONTRACT
  CONTRACT --> LISTENER
  RPC --> LISTENER

  SQL --> BQ
  PS --> BQ
  BQ --> DASH

  SM --> API
  SM --> MCP
  SM --> LISTENER
  SM --> SIGNER
  SM --> INTEG
```
