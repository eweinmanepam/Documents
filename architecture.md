# StableImpact — Architecture (Roadmap-aligned)

Source document: `stableimpact-roadmap-google-cloud-enterprise-en.md`

This document contains:

- High-level architecture overview (3 surfaces + shared backbone)
- Three architecture diagrams:
  1. Agents (Gemini Enterprise + ADK)
  2. External Portal (Nonprofit + Donor)
  3. Reviewer Console (Authenticated human decisioning)
- Requirements split by surface: **External Portal (PR)**, **Agents (AR)**, **Reviewer Console (RR)**
- Explicit boundary: **Reviewer Console vs External Portal**
- Dataflow diagrams (sequence diagrams)
- Agent specification (roles, tool access policy, safety constraints, evaluation criteria)

---

## 1) High-level Architecture Overview

StableImpact is a three-surface system tied together by a private MCP and deterministic backend:

1. **Agents surface (Gemini Enterprise + ADK on Agent Runtime)**
   - Converts unstructured inputs into structured drafts, runs policy/risk/evidence reasoning, and produces **recommendations**.
   - Agents can only act through **StableImpact MCP** (domain tools).
   - Agents **never approve or sign**.

2. **External Portal surface (Nonprofit + Donor)**
   - Handles document uploads (signed URLs → quarantine → sanitize → extract).
   - Handles donor testnet contributions and shows **transaction proof**.
   - No approvals.

3. **Reviewer Console surface (Enterprise reviewer)**
   - The only place where **authenticated human approvals** are recorded.
   - Shows the “decision packet”: risk + evidence highlights + exact release parameters + simulation status.
   - Triggers deterministic orchestration (Pub/Sub/Eventarc → Workflows) that enforces single-use approvals and exact-parameter validation.

Shared backbone:

- **Cloud Run**: MCP, backend API, approval service, EVM listener, controlled signer adapter, integrations adapter
- **Cloud SQL**: authoritative state + append-only ledger
- **Cloud Storage + DLP + Document AI**: evidence pipeline (quarantine → sanitized → extracted fields)
- **Pub/Sub + Workflows**: deterministic execution path after approval
- **Base Sepolia contract + authorized RPC**: verifiable test-USDC contributions/releases
- **BigQuery**: reconciliation/audit/impact views

---

## 2) Architecture Diagrams

### 2.1 Agents Architecture (Gemini Enterprise + ADK)

```mermaid
flowchart TB
  subgraph GE["Gemini Enterprise Agent Surface"]
    USER["Operator / Analyst / (Viewer)"]
    GEM["Gemini Enterprise UI"]
  end

  subgraph AP["Agent Platform"]
    RUNTIME["Agent Runtime"]
    ORCH["ADK Orchestrator"]
    SPEC["Specialist Agents<br/>Campaign | Compliance | Matching | Evidence | Treasury | Audit | Ops"]
    SEARCH["Agent Platform Search<br/>(Policies/Guidance)"]
    SESS["Sessions"]
    ARMOR["Model Armor / Safety Controls"]
  end

  subgraph CR["Cloud Run (Authoritative Services)"]
    MCP["StableImpact MCP<br/>(ONLY authoritative tool interface)"]
    API["Domain Backend API<br/>(state machine, authz, ledger)"]
    INTEG["Enterprise Integrations Adapter<br/>(or Application Integration)"]
  end

  subgraph DATA["Data Plane"]
    SQL["Cloud SQL (Postgres)<br/>(source of truth)"]
    BQ["BigQuery<br/>(audit/reconciliation/metrics)"]
  end

  subgraph CHAIN["Read-only Chain Access (Allowed to Agents)"]
    RPC["Authorized RPC / Read-only provider"]
    CONTRACT["Base Sepolia Contract<br/>(events/state)"]
  end

  USER --> GEM --> RUNTIME --> ORCH
  ORCH --> SPEC
  ORCH <--> SESS
  ARMOR --> ORCH
  ORCH --> SEARCH

  SPEC --> MCP
  MCP --> API
  MCP --> INTEG
  API --> SQL
  SQL --> BQ

  %% Read-only blockchain queries (no execution)
  SPEC --> MCP
  MCP --> RPC --> CONTRACT
```

**Key boundaries**

- Agents talk to the business system **only via MCP**.
- Any chain access from agents is **read-only** (state/tx/receipts/logs) and must be allowlisted.
- Agents must never approve, sign, execute, deploy, or access private keys.

---

### 2.2 External Portal Architecture (Nonprofit + Donor)

```mermaid
flowchart TB
  subgraph USERS["External Users"]
    NP["Nonprofit user"]
    DONOR["Donor (wallet user)"]
  end

  subgraph PORTAL["External Portal (Web)"]
    UI["Portal UI"]
  end

  subgraph CR["Cloud Run Services"]
    API["Domain Backend API"]
    LISTENER["EVM Event Listener<br/>(ingest on-chain events)"]
  end

  subgraph DOCS["Evidence Pipeline"]
    GCSQ["Cloud Storage Quarantine<br/>(restricted)"]
    DLP["Sensitive Data Protection<br/>(redaction)"]
    GCSS["Cloud Storage Sanitized"]
    DOC["Document AI<br/>(extraction)"]
  end

  subgraph EVENTS["Event Plane"]
    PS["Pub/Sub"]
  end

  subgraph DATA["Data"]
    SQL["Cloud SQL (source of truth)"]
    BQ["BigQuery (audit/reconciliation)"]
  end

  subgraph CHAIN["Blockchain (Base Sepolia)"]
    WALLET["Donor wallet"]
    RPC["Authorized RPC"]
    CONTRACT["Escrow Smart Contract<br/>(test-USDC)"]
  end

  %% Nonprofit uploads
  NP --> UI
  UI --> API --> GCSQ
  GCSQ --> DLP --> GCSS --> DOC --> API
  API --> SQL
  API --> PS
  PS --> BQ
  SQL --> BQ

  %% Donor contributions
  DONOR --> UI
  UI --> WALLET --> RPC --> CONTRACT
  CONTRACT --> LISTENER --> PS
  LISTENER --> SQL
```

**Key boundaries**

- Portal supports uploads, status, and donor contributions.
- Portal must **not** be an approval or signing surface.

---

### 2.3 Reviewer Console Architecture (Authenticated approval)

```mermaid
flowchart TB
  subgraph REVIEWERS["Enterprise Reviewer"]
    RUSER["Reviewer (human approver)"]
  end

  subgraph CONSOLE["Reviewer Console (Web)"]
    RUI["Console UI<br/>(decision packet + approval action)"]
  end

  subgraph CR["Cloud Run Services"]
    APPROVAL["Approval Service<br/>(records decisions)"]
    API["Domain Backend API<br/>(validates state/invariants)"]
    SIGNER["Controlled Signer Adapter<br/>(prepare exact tx; no keys in system)"]
    LISTENER["EVM Event Listener"]
  end

  subgraph WFPLANE["Deterministic Orchestration"]
    PS["Pub/Sub / Eventarc"]
    WF["Workflows<br/>(validates approval single-use + parameters, runs simulation, triggers execution path)"]
    SIM["Simulation<br/>(backend or allowlisted provider)"]
  end

  subgraph DATA["Data"]
    SQL["Cloud SQL (source of truth + ledger)"]
    BQ["BigQuery (audit/reconciliation)"]
  end

  subgraph CHAIN["Blockchain (Base Sepolia)"]
    EXW["Human execution wallet<br/>(separate from reviewer identity)"]
    RPC["Authorized RPC"]
    CONTRACT["Escrow Smart Contract"]
  end

  %% Reviewer approval path
  RUSER --> RUI --> APPROVAL --> SQL
  APPROVAL --> PS --> WF
  WF --> API --> SQL

  %% Deterministic execution path
  WF --> SIM
  WF --> SIGNER --> EXW --> RPC --> CONTRACT
  CONTRACT --> LISTENER --> PS
  LISTENER --> SQL
  PS --> BQ
  SQL --> BQ

  %% Console shows status + proofs
  SQL --> RUI
  BQ --> RUI
```

**Key boundaries**

- Console is the **only** surface that can create approvals.
- Execution still requires deterministic workflow validation and a separate human execution wallet signature.

---

## 3) Requirements (split by surface)

### 3.0 Shared global requirements (applies to all surfaces)

**SGR-1 Demo boundary (mandatory)**

- Synthetic organizations/documents/wallets only.
- Testnet-only transactions (Base Sepolia primary).
- Test tokens have no real value; UI/docs must not imply real-funds capability.

**SGR-2 Human authority boundary**

- Agents may recommend; **humans approve**; Workflows enforces deterministic validation; **separate human wallet signs**.

**SGR-3 Security and secrets**

- No secrets in repo, prompts, logs, traces, datasets, or agent memory.
- Provider credentials stored in Secret Manager; only references appear in config.

**SGR-4 Idempotency and anti-duplication**

- All writes require idempotency keys; retries must not create duplicate approvals, events, or disbursements.

**SGR-5 Traceability**

- End-to-end correlation IDs: `trace_id`, `session_id`, `agent_run_id`, `user_id`, `organization_id`, `campaign_id`, `milestone_id`, `approval_id`, `workflow_execution_id`, `transaction_hash`.
- Audit must answer: who approved, what evidence, what amount, where to verify.

---

### 3.1 External Portal requirements (PR)

**PR-1 Authentication & role separation**

- Portal shall support authenticated nonprofit users (and donor session/wallet flow as applicable).
- Portal identities must be distinct from reviewer identities; portal cannot create “approval” records.

**PR-2 Signed uploads + quarantine**

- Portal shall request signed upload URLs and upload documents into a restricted Cloud Storage quarantine area.
- Portal shall enforce constraints: file type allowlist, size limits, clear error handling.

**PR-3 Evidence processing visibility**

- Portal shall display evidence processing status (uploaded → quarantined → sanitized → extracted → ready/failed).
- Portal shall never display raw quarantined content to agents or other users if policy forbids it.

**PR-4 Campaign and milestone context (read-only)**

- Portal shall show campaign milestones and required evidence checklists (as allowed by permissions).

**PR-5 Donor contribution flow (testnet)**

- Portal shall display network, chain ID, token, contract address, and amount with explicit “testnet-only” warnings.
- Portal shall guide wallet contribution submission and show transaction hash + explorer link after contribution.

**PR-6 Milestone evidence submission**

- Portal shall support uploading milestone evidence and linking it to `campaign_id` and `milestone_id`.
- Portal shall show that agent recommendations are pending vs reviewer decision pending.

**PR-7 Mock labeling**

- If any external provider results are mocked (KYC/KYB/sanctions/CRM/ERP), portal shall clearly label mocks and limitations.

**PR must NOT**

- Create or modify approvals.
- Trigger release execution or signing.
- Present agent recommendations as final decisions.

---

### 3.2 Agent requirements (AR)

**AR-1 Orchestration and delegation**

- Orchestrator routes intent, requests missing info, delegates tasks, and stops at human-decision boundaries.

**AR-2 Grounded policy retrieval**

- Agents retrieve policy and guidance from approved sources and cite sources.

**AR-3 Draft campaign creation via MCP**

- Agents create/update/submit campaign drafts using StableImpact MCP tools; backend validates and persists authoritative state.

**AR-4 Sanitized-only evidence consumption**

- Agents consume only sanitized content and Document AI extraction outputs; they do not access quarantined originals if policy forbids.

**AR-5 Risk and due diligence recommendation**

- Compliance agent evaluates versioned risk policy and produces explainable results with uncertainty and escalation.

**AR-6 Milestone evaluation recommendation**

- Evidence agent compares acceptance criteria, budgets, invoices, and verified transactions (read-only chain tools) and outputs a recommendation with citations.

**AR-7 Treasury proposal + simulation request**

- Treasury agent prepares disbursement proposals and requests simulation; cannot execute disbursement.

**AR-8 Audit narrative**

- Audit agent reconciles Cloud SQL with on-chain events and BigQuery views and generates an evidence-backed explanation.

**AR-9 Tooling constraints**

- Agents use StableImpact MCP domain tools only.
- External blockchain MCP is read-only/simulation-only and allowlisted.

**AR must NOT**

- Approve campaigns/disbursements.
- Sign transactions or access private keys/seed phrases.
- Deploy/modify contracts.
- Use generic blockchain write/sign tools.
- Treat model output as authoritative ledger or authoritative on-chain state.

---

### 3.3 Reviewer Console requirements (RR)

**RR-1 Strong authentication**

- Reviewer console requires authenticated access; records reviewer identity with each decision.

**RR-2 Decision context display (campaign and milestone)**
Before approval, console shows:

- campaign/milestone identifiers and state
- evidence list + extracted highlights (sanitized)
- risk assessment + cited policy context
- amount (integer base units), recipient, token, contract, chain ID/network
- agent recommendation labeled non-binding
- simulation status/result
- single-use approval notice

**RR-3 Decision capture**

- Approve/reject/request-info with reason.
- Creates a single-use `approval_id` bound to exact parameters.

**RR-4 Post-approval traceability**

- Shows workflow status and transaction hash + explorer link after execution.

**RR-5 Mock/testnet labeling**

- Clearly labels testnet assets and mocked provider outcomes.

**RR must NOT**

- Contain/access private keys.
- Allow parameter edits after approval without invalidating/re-issuing approval.
- Trigger execution without deterministic workflow validation.

---

### 3.4 Reviewer Console vs External Portal (explicit differences)

1. **Authority:** Portal submits; Console approves (authenticated).
2. **Audience:** Portal = nonprofit/donor; Console = enterprise reviewer.
3. **Information:** Portal = status/tx proof; Console = full decision packet + audit fields.
4. **Security posture:** Portal may be internet-facing; Console must be tightly access-controlled.
5. **Financial controls:** Portal cannot approve/execute; Console records approval, but execution requires Workflows + separate execution wallet signature.

---

## 4) Dataflow Diagrams

### 4.1 Organization onboarding + document processing

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

### 4.2 Campaign creation + approval

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

### 4.3 Contribution + event ingestion

```mermaid
sequenceDiagram
  autonumber
  actor Donor as Donor wallet
  participant Portal as External Portal
  participant RPC as Authorized RPC
  participant Contract as Base Sepolia Contract
  participant Listener as EVM Event Listener (Cloud Run)
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

### 4.4 Milestone evaluation + deterministic disbursement

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

## 5) Agent Specification

### 5.1 Agent roster and responsibilities

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

### 5.2 Tool access control policy (minimum tool allocation)

- Evidence & Impact Agent: **no** `disbursement_execute_approved`
- Treasury Agent: **no** `disbursement_execute_approved`; simulation only
- Orchestrator: may call write tools but should delegate; never request keys
- Only the deterministic Workflow path may lead to “execute-approved” after approval validation.

### 5.3 Shared safety constraints (prompts/callbacks)

- Refuse requests to: approve, sign, transfer, deploy, bypass approval, reveal secrets.
- Cite sources (policy, extracted fields, receipts) for all recommendations.
- Label testnet/mocks; avoid language implying real funds or returns.

### 5.4 Required evaluation criteria (thresholds)

- 0 payments without valid approval
- 0 secrets exposure
- 0 duplicate releases
- ≥90% correct tool choice in primary cases
- ≥90% grounded responses where evaluable
- All critical dataset attacks rejected (prompt injection, malicious docs, secret requests, financial-return promises)

## 6) Per-Agent Requirements + Orchestrator Workflow (Roadmap-aligned)

This section breaks requirements down per agent and defines the orchestrator’s runtime workflow (how it routes intent, uses tools, and enforces boundaries).

### 6.1 Global agent requirements (apply to ALL agents)

**GA-1 Domain-tool-only operations**

- Agents shall perform business operations only through **StableImpact MCP** domain tools.
- Agents shall not call generic vendor APIs directly (enterprise systems, blockchain RPC) except through allowlisted, domain-wrapped mechanisms.

**GA-2 No financial authority**

- Agents shall not approve campaigns or disbursements.
- Agents shall not sign, execute, transfer, deploy, or modify contracts.
- Agents shall not access private keys/seed phrases or any secret material.

**GA-3 Safety + policy compliance**

- Agents shall refuse requests to bypass approval, request secrets, promise returns, or initiate unauthorized financial actions.
- Agents shall label testnet assets and mocked providers as such.

**GA-4 Grounding and citations**

- Agents shall cite sources for recommendations: policy context, extracted fields, on-chain receipts/logs (read-only), and system records.
- Agents shall explicitly state uncertainty and request more information when evidence is missing/ambiguous.

**GA-5 Traceability**

- Each agent run shall include structured correlation identifiers where available: `trace_id`, `session_id`, `agent_run_id`, `user_id`, `organization_id`, `campaign_id`, `milestone_id`, etc.
- Agents shall not store sensitive data in memory; “memory” may only store non-sensitive, consented preferences.

---

### 6.2 Orchestrator Agent — Requirements

**Purpose:** Entry point; routes intent; stops at human decision boundaries; produces unified, grounded responses.

**ORCH-1 Intent classification and routing**

- Orchestrator shall classify user intent into: campaign creation, policy Q&A, evidence upload status, milestone evaluation, treasury proposal, audit/reporting, operations/health, or general help.
- Orchestrator shall delegate specialized work to the appropriate specialist agent.

**ORCH-2 Missing information handling**

- Orchestrator shall request missing required fields before tool calls (e.g., campaign_id, milestone_id).
- Orchestrator shall never “guess” authoritative identifiers or amounts.

**ORCH-3 Human decision boundary enforcement**

- Orchestrator shall stop before any approval/signing/execution steps and instruct the user to use the reviewer console where required.
- Orchestrator shall not attempt to “approve” by writing state changes that simulate an approval.

**ORCH-4 Tooling policy enforcement**

- Orchestrator shall enforce minimum tool allocation per agent (e.g., Evidence agent cannot execute, Treasury agent cannot execute).
- Orchestrator shall fail closed if asked to use unrecognized tools or unknown schema fields.

**ORCH-5 Response composition**

- Orchestrator shall merge sub-agent outputs into a single response that contains:
  - recommendation (if applicable)
  - citations/sources
  - uncertainty notes
  - next action (and who must do it: nonprofit, reviewer, operator)

**Primary tool usage:** Orchestrator generally does not need unique tools; it coordinates specialist tool usage via MCP and uses search/sessions.

---

### 6.3 Program & Campaign Agent — Requirements

**Purpose:** Convert conversations/forms into structured campaigns and milestones; retrieve policy context; persist drafts and submissions.

**PC-1 Campaign drafting**

- Agent shall create and update a campaign draft using MCP.
- Agent shall ensure the campaign includes: objective, budget, milestones (amount + criteria), impact indicators.

**PC-2 Policy contextualization**

- Agent shall retrieve policy context for campaign creation and cite it in explanations.

**PC-3 Validation readiness**

- Agent shall format campaign data to satisfy backend deterministic validation (e.g., integer token base units, required fields).

**PC-4 Safe language**

- Agent shall avoid investment language and financial-return promises.

**Primary MCP tools**

- `campaign_create_draft`
- `campaign_update_draft`
- `campaign_get`
- `campaign_submit`
- `campaign_get_policy_context`
- `campaign_list_eligible` (if also supporting eligibility listing)

---

### 6.4 Compliance & Due-Diligence Agent — Requirements

**Purpose:** Assess eligibility and risk using sanitized documents, extraction outputs, and integration results; produce explainable outcomes.

**CDD-1 Sanitized inputs only**

- Agent shall only consume sanitized document content and extraction outputs.

**CDD-2 Versioned risk policy**

- Agent shall evaluate a versioned risk matrix/policy and cite it.

**CDD-3 Uncertainty and escalation**

- Agent shall classify results as pass/fail/ambiguous and explain what additional evidence is needed for ambiguous cases.

**CDD-4 Provider integration boundaries**

- Agent shall rely on domain tools and integration results (not raw provider APIs).
- Agent shall clearly label mocked provider results.

**Primary MCP tools**

- `organization_get`
- `organization_get_verification`
- `compliance_run_checks`
- `risk_get_policy`
- `risk_save_assessment`

---

### 6.5 Matching Agent — Requirements

**Purpose:** Match eligible campaigns with funding programs and explain mission alignment, constraints, and risks.

**MATCH-1 Eligibility + alignment**

- Agent shall list eligible programs/campaigns and explain alignment to the selected program goals.

**MATCH-2 Non-investment language**

- Agent shall avoid return promises, yield language, “investment” framing, or performance guarantees.

**MATCH-3 Explainability**

- Agent shall cite policies/criteria that drive matching.

**Primary MCP tools**

- `campaign_list_eligible`
- `campaign_get_policy_context`

---

### 6.6 Evidence & Impact Agent — Requirements

**Purpose:** Compare milestone evidence with acceptance criteria, budgets, invoices, and verified transactions; output a sourced recommendation.

**EVI-1 Evidence comparison**

- Agent shall compare extracted document fields (Document AI outputs) against:
  - milestone acceptance criteria
  - approved budget lines
  - prior contributions/releases (read-only chain facts)
- Agent shall identify inconsistencies and missing items.

**EVI-2 Recommendation output**

- Agent shall produce one of: approve / reject / request-more-info.
- Agent shall include citations and an uncertainty statement when needed.

**EVI-3 No state transition authority**

- Agent shall not approve; it may only save an agent review record.

**Primary MCP tools**

- `evidence_create_upload` (if evidence upload initiation is agent-driven; otherwise portal-driven)
- `evidence_get_processing_status`
- `evidence_get_extracted_fields`
- `evidence_get_sanitized_content`
- `milestone_save_agent_review`
- Read-only chain tools:
  - `chain_get_network`
  - `chain_get_campaign_state`
  - `chain_get_token_balance`
  - `chain_get_transaction`
  - `chain_list_contract_events`

---

### 6.7 Treasury Agent — Requirements

**Purpose:** Prepare disbursement proposals and request simulation; explain amount/recipient/conditions; never approve/sign/execute.

**TRE-1 Proposal preparation**

- Agent shall create a disbursement proposal bound to campaign_id + milestone_id + amount + recipient + token + contract + chain_id.

**TRE-2 Simulation request**

- Agent shall request simulation via domain tools and interpret results for the reviewer.

**TRE-3 Refusal of execution**

- Agent shall refuse any attempt to execute, sign, or bypass approvals.

**Primary MCP tools**

- `contribution_prepare`
- `disbursement_create_proposal`
- `disbursement_simulate`
- (Explicitly NOT allowed for this agent): `disbursement_execute_approved`

---

### 6.8 Audit Agent — Requirements

**Purpose:** Reconcile Cloud SQL, chain events, Pub/Sub/Workflow events, and BigQuery; produce traceable audit explanations.

**AUD-1 Reconciliation**

- Agent shall compare operational ledger vs on-chain events and detect discrepancies.

**AUD-2 Audit explanations**

- Agent shall generate explanations answering:
  - who approved
  - what evidence was used
  - what amount was released
  - where the transaction is verifiable (explorer link)
  - whether reconciliation passed

**AUD-3 Dataset generation**

- Agent shall generate or refresh audit datasets/views (where implemented).

**Primary MCP tools**

- `audit_get_campaign_ledger`
- `audit_compare_sources`
- `audit_get_trace`
- `audit_generate_dataset`
- `audit_save_report`
- Read-only chain tools as needed

---

### 6.9 Operations Agent — Requirements

**Purpose:** Summarize system health and failures; recommend actions without changing infrastructure.

**OPS-1 Health reporting**

- Agent shall retrieve service health and failure queues and summarize them in business-meaningful terms.

**OPS-2 Incident reporting**

- Agent shall prepare an incident report that is redacted (no secrets) and includes correlation IDs.

**OPS-3 No mutation**

- Agent shall not modify infrastructure, IAM, deployments, or secrets.

**Primary MCP tools**

- `ops_get_service_health`
- `ops_get_failed_events`
- `ops_get_agent_trace`
- `ops_prepare_incident_report`

---

## 6.10 Orchestrator Workflow (runtime sequence)

### 6.10.1 High-level state machine (what the orchestrator runs)

```mermaid
stateDiagram-v2
  [*] --> Intake
  Intake --> ClassifyIntent
  ClassifyIntent --> GatherContext
  GatherContext --> Delegate

  Delegate --> Compose
  Delegate --> RequestMissing : missing required fields
  RequestMissing --> Intake

  Compose --> HumanBoundaryCheck
  HumanBoundaryCheck --> Refuse : approval/signing/execution requested
  HumanBoundaryCheck --> Respond : safe and complete

  Refuse --> [*]
  Respond --> [*]
```
