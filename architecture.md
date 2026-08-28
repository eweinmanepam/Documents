I can’t directly write to your GitHub repo from this chat, but I can give you a **ready-to-commit `architecture.md`** and the exact steps to add it to `https://github.com/eweinmanepam/Documents`.

## How to save it into your repo (quick steps)

1. Open: `https://github.com/eweinmanepam/Documents`
2. Click **Add file → Create new file**
3. In **Filename**, enter: `architecture.md`
4. Paste the Markdown content below
5. Click **Commit changes**

---

# `architecture.md` (paste this entire content)

````markdown
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
````

---

## 2) Requirements

### 2.1 Functional requirements (FR)

**FR-1 Campaign creation (Gemini Enterprise + MCP)**

- System shall allow an operator to create a structured campaign via Gemini Enterprise, with agent-assisted collection of missing info.
- System shall persist drafts and final submissions through StableImpact MCP domain tools.

**FR-2 Document intake + sanitization**

- System shall provide signed upload URLs and quarantine incoming documents.
- System shall sanitize documents using Sensitive Data Protection (or explicitly documented equivalent) before any agent consumption.

**FR-3 Document extraction**

- System shall extract structured fields via Document AI (OCR/forms/invoices) and store results linked to organization/campaign/milestone.

**FR-4 Compliance/risk recommendation**

- System shall run policy checks and produce an evidence-cited risk assessment with uncertainty handling.

**FR-5 Contribution intake (Base Sepolia)**

- System shall display network/token/contract and allow a donor to contribute test-USDC.
- System shall ingest `ContributionReceived` events via an EVM listener and persist idempotently.

**FR-6 Milestone evidence evaluation**

- System shall accept milestone evidence uploads, sanitize and extract them, and produce a grounded recommendation (approve/reject/request info).

**FR-7 Human approval (mandatory)**

- System shall provide an authenticated reviewer console to record decisions.
- System shall treat approval as **single-use** and bind it to campaign+milestone+amount+recipient+token+contract+chain_id.

**FR-8 Deterministic disbursement**

- System shall use Workflows to validate deterministic rules, request simulation, and only then trigger controlled execution.

**FR-9 Controlled signing**

- System shall require a separate human execution wallet to sign the exact approved transaction.
- System shall ensure agents never receive private keys or generic signing capability.

**FR-10 Reconciliation + audit explanation**

- System shall reconcile Cloud SQL state and on-chain events in BigQuery.
- System shall produce an audit/impact explanation answering: who approved, what evidence, what amount, where to verify.

**FR-11 Enterprise integration (at least one visible)**

- System shall implement at least one real enterprise integration, or a contract-compatible mock clearly labeled.

### 2.2 Non-functional requirements (NFR)

- **NFR-1 Security/IAM:** least privilege, separate identities, private Cloud Run by default.
- **NFR-2 Data protection:** no secrets in repo/logs/prompts/datasets/traces; Secret Manager references only.
- **NFR-3 Traceability:** end-to-end correlation IDs: `trace_id, session_id, agent_run_id, campaign_id, milestone_id, approval_id, workflow_execution_id, transaction_hash`, etc.
- **NFR-4 Idempotency:** all write operations require idempotency keys; retries must not create duplicates.
- **NFR-5 Observability:** structured logs, traces, alerts for failures/rejections/reconciliation differences; exclude full prompts/docs unless explicitly approved.
- **NFR-6 Reliability:** core path must survive auxiliary MCP failure if authorized RPC remains available.
- **NFR-7 Demo boundary:** synthetic data only; testnet only; mock labeling enforced.

### 2.3 Explicit prohibitions (“must-not”)

- Agents must not approve disbursements, sign transactions, deploy contracts, or execute arbitrary SQL/shell.
- MCP must not expose generic blockchain write tools (`sendTokens`, `writeContract`, `sendTransactions`, etc.).

---

## 3) Dataflow Diagrams

### 3.1 Organization onboarding + document processing

```mermaid
sequenceDiagram
  autonumber
  actor OrgUser as Nonprofit user
  participant Portal as External Portal
  participant API as Backend API (Cloud Run)
  participant GCSQ as GCS Quarantine
  participant DLP as Sensitive Data Protection
  participant GCSS as GCS Sanitized
  participant DocAI as Document AI
  participant PS as Pub/Sub
  participant SQL as Cloud SQL

  OrgUser->>Portal: Authenticate + request upload
  Portal->>API: Request signed upload URL
  API-->>Portal: Signed URL + upload constraints
  OrgUser->>GCSQ: Upload document (quarantine)
  API->>DLP: Trigger inspection/redaction (async)
  DLP->>GCSS: Write sanitized copy + redaction report
  API->>DocAI: Submit sanitized doc for extraction
  DocAI-->>API: Extracted fields/artifacts
  API->>SQL: Persist DocumentExtraction + hashes + links
  API->>PS: Publish organization.documents.processed
```

### 3.2 Campaign creation + approval

```mermaid
sequenceDiagram
  autonumber
  actor Operator as Operator (Gemini Enterprise)
  participant GE as Gemini Enterprise
  participant ORCH as ADK Orchestrator
  participant SEARCH as Agent Platform Search
  participant MCP as StableImpact MCP
  participant API as Backend API
  participant Review as Reviewer Console
  participant Approval as Approval Service
  participant SQL as Cloud SQL
  participant PS as Pub/Sub

  Operator->>GE: Start "Create campaign"
  GE->>ORCH: User intent + context
  ORCH->>SEARCH: Retrieve policies/guidance
  SEARCH-->>ORCH: Policy excerpts + citations
  ORCH->>MCP: campaign_create_draft(...)
  MCP->>API: Create draft (validated)
  API->>SQL: Persist draft campaign
  ORCH->>MCP: campaign_submit(...)
  MCP->>API: Submit campaign (state transition)
  API->>SQL: Persist submitted campaign

  Operator->>Review: Reviewer logs in and reviews
  Review->>Approval: Record decision + reason (authenticated)
  Approval->>SQL: Persist Approval (single-use)
  Approval->>PS: Emit campaign.approved (or rejected)
```

### 3.3 Contribution + event ingestion

```mermaid
sequenceDiagram
  autonumber
  actor Donor as Donor wallet
  participant Portal as External Portal
  participant RPC as Authorized RPC
  participant Contract as Base Sepolia Contract
  participant Listener as EVM Listener (Cloud Run)
  participant PS as Pub/Sub
  participant SQL as Cloud SQL
  participant BQ as BigQuery

  Portal-->>Donor: Show network/token/contract/amount + warnings
  Donor->>RPC: Submit contribution tx
  RPC->>Contract: Execute deposit
  Contract-->>RPC: Emit ContributionReceived
  Listener->>RPC: Poll/subscribe logs (allowlisted)
  Listener->>PS: Publish contribution.confirmed (normalized)
  Listener->>SQL: Persist contribution + blockchain event (idempotent)
  PS->>BQ: Stream normalized event
  SQL->>BQ: Replicate operational records (batch/stream)
```

### 3.4 Milestone evaluation + deterministic disbursement

```mermaid
sequenceDiagram
  autonumber
  actor OrgUser as Nonprofit user
  actor Reviewer as Human reviewer
  participant ORCH as ADK Orchestrator
  participant MCP as StableImpact MCP
  participant API as Backend API
  participant Approval as Approval Service
  participant PS as Pub/Sub/Eventarc
  participant WF as Workflows
  participant Sim as Simulation (backend or read-only provider)
  participant Signer as Controlled signer adapter
  participant Wallet as Human execution wallet
  participant RPC as Authorized RPC
  participant Contract as Smart contract
  participant Listener as EVM listener
  participant SQL as Cloud SQL
  participant BQ as BigQuery

  OrgUser->>API: Upload milestone evidence (portal)
  API->>SQL: Link evidence to milestone

  ORCH->>MCP: evidence_get_extracted_fields + policy context
  MCP->>API: Fetch sanitized/extracted evidence + criteria
  API-->>MCP: Structured data

  ORCH->>MCP: milestone_save_agent_review(recommendation + citations)
  MCP->>API: Persist recommendation
  API->>SQL: Save agent review (non-authoritative)

  Reviewer->>Approval: Approve milestone release (authenticated)
  Approval->>PS: Emit disbursement.approved (includes approval_id)

  PS->>WF: Trigger workflow execution
  WF->>API: Validate approval single-use + state + invariants
  WF->>Sim: Request transaction simulation
  Sim-->>WF: Simulation OK + exact calldata
  WF->>Signer: Prepare exact tx payload (no private keys)
  Signer-->>WF: Tx payload for signing
  WF->>Wallet: Request human signature (out-of-band)
  Wallet->>RPC: Submit signed tx
  RPC->>Contract: Execute release
  Contract-->>RPC: Emit FundsReleased

  Listener->>RPC: Fetch receipt/logs
  Listener->>PS: Publish disbursement.confirmed
  Listener->>SQL: Persist disbursement + blockchain event (idempotent)
  SQL->>BQ: Update operational + audit datasets
  PS->>BQ: Update normalized events
```

---

## 4) Agent Specification

### 4.1 Agent roster and responsibilities

| Agent                            | Primary responsibilities                                                                                     | Must-not boundaries                           | Primary MCP tools                                                                                                       |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------ | --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Orchestrator                     | Routes intent, asks for missing fields, delegates, enforces “stop at human decision boundary”                | Must not approve or execute financial actions | All domain tools as needed; prefer delegating writes                                                                    |
| Program & Campaign Agent         | Draft campaigns, milestones, impact indicators; retrieve policy context; save drafts                         | Must not “auto-approve”                       | `campaign_create_draft`, `campaign_update_draft`, `campaign_submit`, `campaign_get_policy_context`                      |
| Compliance & Due-Diligence Agent | Review sanitized org data + extractions + integration results; produce risk assessment with uncertainty      | Must not change approval state                | `organization_get`, `organization_get_verification`, `compliance_run_checks`, `risk_get_policy`, `risk_save_assessment` |
| Matching Agent                   | Match campaigns to programs; explain alignment and risk                                                      | Must avoid investment language/returns        | `campaign_list_eligible`, `campaign_get_policy_context`                                                                 |
| Evidence & Impact Agent          | Compare milestone evidence to criteria/budget/invoices/on-chain facts; recommend approve/reject/request info | Must not execute disbursement                 | `evidence_get_*`, `chain_get_*`, `milestone_save_agent_review`                                                          |
| Treasury Agent                   | Prepare disbursement proposal; request simulation; explain amount/recipient/conditions                       | Must not approve/sign/execute                 | `chain_get_token_balance`, `chain_get_campaign_state`, `disbursement_create_proposal`, `disbursement_simulate`          |
| Audit Agent                      | Reconcile sources; detect discrepancies; generate audit dataset/report                                       | Must not mutate financial state               | `audit_get_campaign_ledger`, `audit_compare_sources`, `audit_generate_dataset`, `audit_save_report`                     |
| Operations Agent                 | Summarize health, failures, traces; recommend actions                                                        | Must not modify infrastructure                | `ops_get_service_health`, `ops_get_failed_events`, `ops_get_agent_trace`, `ops_prepare_incident_report`                 |

### 4.2 Tool access control policy (minimum tool allocation)

- Evidence & Impact Agent: **no** `disbursement_execute_approved`
- Treasury Agent: **no** `disbursement_execute_approved`; simulation only
- Orchestrator: may call write tools but should delegate; never request keys
- Only the deterministic Workflow path may lead to “execute-approved” after approval validation.

### 4.3 Shared safety constraints (prompts/callbacks)

- Refuse requests to: approve, sign, transfer, deploy, bypass approval, reveal secrets.
- Cite sources (policy, extracted fields, receipts) for all recommendations.
- Label testnet/mocks; avoid language implying real funds or returns.

### 4.4 Required evaluation criteria (thresholds)

- 0 payments without valid approval
- 0 secrets exposure
- 0 duplicate releases
- ≥90% correct tool choice in primary cases
- ≥90% grounded responses where evaluable
- All critical dataset attacks rejected (prompt injection, malicious docs, secret requests, financial-return promises)

```

If you’d like, I can also generate a short `README.md` snippet explaining how to render Mermaid diagrams on GitHub (and alternatives if GitHub doesn’t render them in your view).
```
