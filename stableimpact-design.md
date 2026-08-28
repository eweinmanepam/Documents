# Design Document: StableImpact

_Drafted 2026-08-28._

## Application Description

StableImpact is an agentic funding and impact-assurance platform for organizations that
manage grants, donations, corporate social responsibility programs, humanitarian
funding, or international development programs. Gemini-based agents structure funding
applications, review policy and evidence, assess risk, and recommend milestone-based
disbursements, but a human always makes the actual approval decision. Once a decision is
recorded, deterministic Google Cloud workflows execute it, and a Base Sepolia smart
contract provides a verifiable (testnet) record of the USDC contribution or release. The
platform's job is to connect funding decisions, evidence, approvals, payments,
reconciliation, and impact reporting into one traceable lifecycle, rather than leaving
them scattered across disconnected systems.

Several distinct kinds of users interact with the platform. **Beneficiary
organizations** (nonprofits, grantees) apply for funding, submitting organizational
documents, a budget, and an impact plan, and later submit evidence against individual
milestones. **Donors/funders** contribute test USDC to a campaign. **Business reviewers**
— authenticated humans — review the agents' risk and evidence recommendations and record
the actual approve/reject decision for both campaigns and disbursements; a separate
**execution wallet holder** signs the on-chain release itself, only after that decision
has passed deterministic validation, and is treated as a distinct piece of evidence even
when the same person holds both roles. **Auditors and operators** consume reconciliation
and impact reporting without acting inside the flow. Finally, the **Gemini/ADK agents**
themselves — an orchestrator plus specialists for compliance, matching, evidence/impact,
treasury, audit, and operations — are a distinct actor class: they read, recommend, and
explain, but they never approve, sign, or execute anything themselves.

## Architecture

### High-level system architecture

```mermaid
flowchart LR
  subgraph Public[Public internet and user channels]
    H1_GE[Gemini Enterprise UI]
    H1_PORTAL[External portal]
    H1_REVIEW[Reviewer console]
    H1_DASH[Audit and impact dashboard]
    H1_DONORW[Donor wallet]
  end

  subgraph Platform[Google managed Agent Platform]
    H1_RUNTIME[Agent runtime]
    H1_ORCH[ADK orchestrator]
    H1_AGENTS[Specialist agents]
    H1_SEARCH[Platform search]
    H1_ARMOR[Model armor]
  end

  H1_GE -->|TLS| H1_RUNTIME
  H1_RUNTIME -->|internal| H1_ORCH
  H1_ORCH -->|internal| H1_AGENTS
  H1_ORCH -->|internal| H1_SEARCH
  H1_ARMOR -->|policy| H1_ORCH

  H1_PORTAL -->|HTTPS TLS| H1_REVIEW
  H1_DASH -->|HTTPS TLS| H1_REVIEW
  H1_DONORW -->|browser| H1_PORTAL
```

```mermaid
flowchart LR
  subgraph AppSvc[Org GCP app services]
    H2_MCP[StableImpact MCP]
    H2_API[Domain backend]
    H2_APPROVAL[Approval service]
    H2_LISTENER[EVM event listener]
    H2_SIGNER[Signer adapter]
  end

  subgraph DataZone[Org GCP data and evidence]
    H2_SQL[Cloud SQL]
    H2_GCS[Cloud Storage]
    H2_DLP[Sensitive data protection]
    H2_DOC[Document AI]
    H2_BQ[BigQuery]
  end

  subgraph EventZone[Org GCP event plane]
    H2_PS[Pub Sub]
    H2_WF[Workflows]
  end

  H2_MCP -->|HTTPS mTLS| H2_API
  H2_API -->|TLS| H2_SQL
  H2_API -->|HTTPS TLS| H2_GCS
  H2_GCS -->|HTTPS TLS| H2_DLP
  H2_DLP -->|HTTPS TLS| H2_DOC
  H2_DOC -->|TLS| H2_SQL

  H2_API -->|Pub Sub TLS| H2_PS
  H2_APPROVAL -->|TLS| H2_SQL
  H2_APPROVAL -->|Pub Sub TLS| H2_PS

  H2_LISTENER -->|Pub Sub TLS| H2_PS
  H2_PS -->|Eventarc| H2_WF
  H2_WF -->|HTTPS mTLS| H2_SIGNER

  H2_SQL -->|BigQuery TLS| H2_BQ
  H2_PS -->|BigQuery TLS| H2_BQ
```

```mermaid
flowchart LR
  subgraph Chain[Base Sepolia testnet]
    H3_RPC[Authorized RPC]
    H3_CONTRACT[Escrow contract]
    H3_WALLET[Execution wallet]
  end

  subgraph Offchain[Off chain execution]
    H3_WF[Workflows]
    H3_LISTENER[EVM listener]
  end

  H3_WF -->|HTTPS mTLS| H3_WALLET
  H3_WALLET -->|JSON RPC HTTPS| H3_RPC
  H3_RPC -->|EVM call| H3_CONTRACT
  H3_CONTRACT -->|event| H3_LISTENER
  H3_LISTENER -->|JSON RPC HTTPS| H3_RPC
```

### Portal

```mermaid
flowchart LR
  subgraph Public[Public internet]
    P_ORG[Beneficiary org user]
    P_DONOR[Donor]
    P_PORTAL[External portal]
    P_DONORW[Donor wallet]
  end

  subgraph AppSvc[Application services]
    P_API[Domain backend]
  end

  subgraph DataZone[Data and evidence]
    P_GCS[Cloud Storage quarantine]
  end

  subgraph Chain[Base Sepolia testnet]
    P_RPC[Authorized RPC]
    P_CONTRACT[Escrow contract]
  end

  P_ORG -->|HTTPS TLS| P_PORTAL
  P_DONOR -->|HTTPS TLS| P_PORTAL
  P_PORTAL -->|HTTPS TLS OIDC| P_API
  P_PORTAL -->|Signed URL HTTPS TLS| P_GCS
  P_PORTAL -->|wallet connect| P_DONORW
  P_DONORW -->|JSON RPC HTTPS| P_RPC
  P_RPC -->|EVM tx| P_CONTRACT
```

The portal never touches a private key — the donor's own wallet signs and submits the
contribution directly to the chain. The portal only assembles and displays the
transaction parameters, then later reads back confirmed contribution state from the
backend once the listener has ingested the on-chain event.

### Reviewer console

```mermaid
flowchart LR
  subgraph Public[Public internet authenticated]
    R_REVIEWER[Business reviewer]
    R_CONSOLE[Reviewer console]
  end

  subgraph AppSvc[Application services]
    R_API[Domain backend]
    R_APPROVAL[Approval service]
  end

  subgraph DataZone[Data and evidence]
    R_SQL[Cloud SQL]
  end

  subgraph EventZone[Event plane]
    R_PS[Pub Sub]
    R_WF[Workflows]
  end

  R_REVIEWER -->|SSO OIDC| R_CONSOLE
  R_CONSOLE -->|HTTPS TLS OIDC| R_API
  R_API -->|TLS| R_SQL
  R_CONSOLE -->|HTTPS TLS OIDC| R_APPROVAL
  R_APPROVAL -->|TLS| R_SQL
  R_APPROVAL -->|Pub Sub TLS| R_PS
  R_PS -->|Eventarc| R_WF
```

The console only records the business decision — it never signs a blockchain
transaction. The execution wallet is invoked separately, from Workflows, only after
deterministic validation has passed.

### Agents

Tool-to-agent mappings below are inferred from each agent's responsibility description
in the source roadmap and the roadmap's flat MCP tool catalog, since the source material
does not explicitly assign tools to agents. Confidence is lowest for the **Matching**
agent, whose tool set is not clued by the source text at all; see Open Questions.

**Orchestrator agent** — calls no MCP tools directly; it routes intent and delegates to
specialists.

```mermaid
flowchart LR
  subgraph Public[Public]
    OA_GE[Gemini Enterprise UI]
  end

  subgraph Platform[Agent Platform]
    OA_RUNTIME[Agent runtime]
    OA_ORCH[Orchestrator agent]
    OA_SEARCH[Platform search]
    OA_SPECIALISTS[Specialist agents]
    OA_ARMOR[Model armor]
  end

  OA_GE -->|TLS| OA_RUNTIME
  OA_RUNTIME -->|internal| OA_ORCH
  OA_ARMOR -->|policy| OA_ORCH
  OA_ORCH -->|internal| OA_SEARCH
  OA_ORCH -->|internal| OA_SPECIALISTS
```

**Program & campaign agent**

```mermaid
flowchart LR
  subgraph Platform[Agent Platform]
    PC_AGENT[Program and campaign agent]
    PC_SEARCH[Platform search]
  end

  subgraph AppSvc[Application services]
    PC_MCP[StableImpact MCP]
    PC_API[Domain backend]
  end

  subgraph DataZone[Data zone]
    PC_SQL[Cloud SQL]
  end

  PC_AGENT -->|internal| PC_SEARCH
  PC_AGENT -->|MCP HTTPS mTLS| PC_MCP
  PC_MCP -->|HTTPS mTLS| PC_API
  PC_API -->|TLS| PC_SQL
```

**Compliance & due-diligence agent**

```mermaid
flowchart LR
  subgraph Platform[Agent Platform]
    C_AGENT[Compliance and due diligence agent]
  end

  subgraph AppSvc[Application services]
    C_MCP[StableImpact MCP]
    C_API[Domain backend]
    C_INTEG[Integration connector]
  end

  subgraph DataZone[Data zone]
    C_SQL[Cloud SQL]
  end

  C_AGENT -->|MCP HTTPS mTLS| C_MCP
  C_MCP -->|HTTPS mTLS| C_API
  C_API -->|HTTPS TLS| C_INTEG
  C_API -->|TLS| C_SQL
```

**Matching agent** _(lowest-confidence tool inference — see Open Questions)_

```mermaid
flowchart LR
  subgraph Platform[Agent Platform]
    M_AGENT[Matching agent]
  end

  subgraph AppSvc[Application services]
    M_MCP[StableImpact MCP]
    M_API[Domain backend]
  end

  subgraph DataZone[Data zone]
    M_SQL[Cloud SQL]
  end

  M_AGENT -->|MCP HTTPS mTLS| M_MCP
  M_MCP -->|HTTPS mTLS| M_API
  M_API -->|TLS| M_SQL
```

**Evidence & impact agent**

```mermaid
flowchart LR
  subgraph Platform[Agent Platform]
    E_AGENT[Evidence and impact agent]
  end

  subgraph AppSvc[Application services]
    E_MCP[StableImpact MCP]
    E_API[Domain backend]
  end

  subgraph DataZone[Data and evidence]
    E_SQL[Cloud SQL]
    E_GCS[Cloud Storage sanitized]
  end

  subgraph Chain[Chain read only]
    E_RPC[Authorized RPC]
  end

  E_AGENT -->|MCP HTTPS mTLS| E_MCP
  E_MCP -->|HTTPS mTLS| E_API
  E_API -->|TLS| E_SQL
  E_API -->|HTTPS TLS| E_GCS
  E_MCP -->|JSON RPC HTTPS| E_RPC
```

**Treasury agent** — has no access to `disbursement_execute_approved`; execution is
invoked only from Workflows after a valid human decision, per the MCP rules.

```mermaid
flowchart LR
  subgraph Platform[Agent Platform]
    T_AGENT[Treasury agent]
  end

  subgraph AppSvc[Application services]
    T_MCP[StableImpact MCP]
    T_API[Domain backend]
  end

  subgraph DataZone[Data zone]
    T_SQL[Cloud SQL]
  end

  subgraph Chain[Chain read only]
    T_RPC[Authorized RPC]
  end

  T_AGENT -->|MCP HTTPS mTLS| T_MCP
  T_MCP -->|HTTPS mTLS| T_API
  T_API -->|TLS| T_SQL
  T_MCP -->|JSON RPC HTTPS| T_RPC
```

**Audit agent**

```mermaid
flowchart LR
  subgraph Platform[Agent Platform]
    A_AGENT[Audit agent]
  end

  subgraph AppSvc[Application services]
    A_MCP[StableImpact MCP]
    A_API[Domain backend]
  end

  subgraph DataZone[Data and evidence]
    A_SQL[Cloud SQL]
    A_BQ[BigQuery]
  end

  subgraph Chain[Chain read only]
    A_RPC[Authorized RPC]
  end

  A_AGENT -->|MCP HTTPS mTLS| A_MCP
  A_MCP -->|HTTPS mTLS| A_API
  A_API -->|TLS| A_SQL
  A_API -->|BigQuery TLS| A_BQ
  A_MCP -->|JSON RPC HTTPS| A_RPC
```

**Operations agent**

```mermaid
flowchart LR
  subgraph Platform[Agent Platform]
    O_AGENT[Operations agent]
  end

  subgraph AppSvc[Application services]
    O_MCP[StableImpact MCP]
  end

  subgraph ObsZone[Observability]
    O_LOG[Cloud Logging]
  end

  O_AGENT -->|MCP HTTPS mTLS| O_MCP
  O_MCP -->|API TLS| O_LOG
```

### Orchestrator delegation workflow

The diagram below shows how the orchestrator sequences work across specialists over a
full campaign lifecycle, and — critically — where it stops for a human decision rather
than carrying a request across an approval gate itself.

```mermaid
sequenceDiagram
  participant User as User - Gemini Enterprise
  participant ORCH as Orchestrator
  participant CAMPAIGN as Campaign agent
  participant COMPLIANCE as Compliance agent
  participant MATCH as Matching agent
  participant EVIDENCE as Evidence agent
  participant TREASURY as Treasury agent
  participant AUDIT as Audit agent
  participant OPS as Operations agent
  participant MCP as StableImpact MCP
  participant REVIEWER as Reviewer - console, outside agent flow

  User->>ORCH: Create clean-water campaign for Org X
  ORCH->>CAMPAIGN: delegate: structure application
  CAMPAIGN->>MCP: campaign_create_draft / campaign_update_draft
  MCP-->>CAMPAIGN: draft saved
  CAMPAIGN-->>ORCH: draft ready

  ORCH->>COMPLIANCE: delegate: assess risk/eligibility
  COMPLIANCE->>MCP: organization_get, compliance_run_checks, risk_get_policy
  MCP-->>COMPLIANCE: checks + policy
  COMPLIANCE->>MCP: risk_save_assessment
  COMPLIANCE-->>ORCH: risk assessment + citations

  ORCH->>MATCH: delegate: confirm program alignment
  MATCH->>MCP: campaign_list_eligible, campaign_get_policy_context
  MCP-->>MATCH: eligible programs
  MATCH-->>ORCH: alignment explanation

  ORCH-->>User: sourced campaign recommendation
  Note over ORCH,User: Human-decision boundary - orchestrator stops
  User->>REVIEWER: (separately, via reviewer console) approves campaign

  Note over REVIEWER: Contribution + milestone evidence submission happen outside the agent flow

  User->>ORCH: Review milestone 1 evidence
  ORCH->>EVIDENCE: delegate: compare evidence vs criteria
  EVIDENCE->>MCP: evidence_get_extracted_fields, evidence_get_sanitized_content, chain_get_transaction
  MCP-->>EVIDENCE: extracted data + verified tx
  EVIDENCE->>MCP: milestone_save_agent_review
  EVIDENCE-->>ORCH: recommendation (approve/reject/more info) + sources

  ORCH->>TREASURY: delegate: prepare disbursement proposal
  TREASURY->>MCP: chain_get_campaign_state, disbursement_create_proposal, disbursement_simulate
  MCP-->>TREASURY: simulated proposal
  TREASURY-->>ORCH: proposal + simulation result

  ORCH-->>User: milestone recommendation + disbursement proposal
  Note over ORCH,User: Human-decision boundary - orchestrator stops
  User->>REVIEWER: (separately) approves disbursement

  Note over REVIEWER: Workflows validates, human wallet signs, contract releases - outside agent flow

  User->>ORCH: Why was this payment released?
  ORCH->>AUDIT: delegate: reconcile & explain
  AUDIT->>MCP: audit_get_campaign_ledger, audit_compare_sources, audit_get_trace
  MCP-->>AUDIT: reconciled ledger + trace
  AUDIT-->>ORCH: sourced explanation
  ORCH-->>User: evidence-backed audit answer

  User->>ORCH: Any failed events or incidents?
  ORCH->>OPS: delegate: check service health
  OPS->>MCP: ops_get_service_health, ops_get_failed_events
  MCP-->>OPS: health/incident summary
  OPS-->>ORCH: summary (no secrets exposed)
  ORCH-->>User: operations summary
```

## Requirements / Use Cases

### Use Case 1: Organization onboarding

An organization applying for funding for the first time must be identified and screened
before any campaign work begins. A **beneficiary organization user** authenticates
(account provisioning itself is out of scope for this use case — see Assumptions and
Constraints, and the related Open Question), then submits identifying and organizational
documents through the **portal**. Those documents are quarantined, inspected and redacted
by **Sensitive Data Protection**, and structured by **Document AI**, so that only
sanitized, structured data ever reaches an agent. The **compliance agent** reads that
sanitized case file, runs KYC/KYB and sanctions checks against a real provider or a
clearly labeled mock, and produces a sourced risk summary. A **human reviewer** then
makes the actual onboarding decision — the agent never decides admission on its own.

```mermaid
sequenceDiagram
  participant USER as Org user
  participant IDP as Identity Platform
  participant PORTAL as Portal
  participant GCS as Cloud Storage - quarantine
  participant DLP as Sensitive Data Protection
  participant DOC as Document AI
  participant SQL as Cloud SQL
  participant PS as Pub/Sub
  participant COMPLIANCE as Compliance agent
  participant INTEG as Application Integration
  participant REVIEWER as Human reviewer

  USER->>IDP: authenticate
  IDP-->>PORTAL: identity confirmed
  PORTAL->>GCS: signed upload URL, files uploaded
  GCS->>DLP: inspect/redact
  DLP->>DOC: extract structured fields
  DOC->>SQL: normalized case file
  SQL->>PS: organization.documents.processed
  PS-->>COMPLIANCE: notify (via orchestrator delegation)
  COMPLIANCE->>SQL: read sanitized case file
  COMPLIANCE->>INTEG: run KYC/KYB & sanctions checks (real or labeled mock)
  INTEG-->>COMPLIANCE: check results
  COMPLIANCE-->>REVIEWER: sourced risk summary
  REVIEWER-->>SQL: onboarding decision recorded
```

### Use Case 2: Campaign creation and approval

Once onboarded, an organization works with **Gemini Enterprise** to turn a funding need
into a structured campaign. The **program & campaign agent** retrieves the applicable
eligibility policy from **Agent Platform Search**, drafts the campaign through the MCP
server, and the **domain backend** validates its objectives, budget, milestones, and
amounts before anything is presented for approval. The campaign and compliance agents
jointly produce a sourced recommendation, and a **human reviewer** either approves the
campaign or sends it back for changes — the backend then records that decision and
publishes it as an event other components can react to.

```mermaid
sequenceDiagram
  participant USER as Applicant
  participant GE as Gemini Enterprise
  participant SEARCH as Agent Platform Search
  participant CAMPAIGN as Campaign agent
  participant COMPLIANCE as Compliance agent
  participant MCP as StableImpact MCP
  participant API as Domain backend
  participant REVIEWER as Human reviewer
  participant PS as Pub/Sub

  USER->>GE: describe campaign
  GE->>SEARCH: retrieve eligibility policy
  GE->>CAMPAIGN: delegate: structure campaign
  CAMPAIGN->>MCP: campaign_create_draft
  MCP->>API: validate objectives/budget/milestones/amounts
  CAMPAIGN->>COMPLIANCE: request risk input
  COMPLIANCE-->>CAMPAIGN: sourced risk assessment
  CAMPAIGN-->>REVIEWER: recommendation for approval
  REVIEWER->>API: approve / request changes
  API->>PS: campaign.approved
```

### Use Case 3: Test-USDC contribution

A **donor** funds an approved campaign directly on-chain. The portal shows the donor the
exact network, token, contract, and amount before anything is signed, then the donor's
own wallet signs and submits the contribution — the platform never holds the donor's
key. The **EVM listener** watches the Base Sepolia contract, confirms the event actually
belongs to the expected chain, contract, and block, and only then lets the contribution
enter the system of record, where it is recorded exactly once even if the underlying
blockchain event is redelivered.

```mermaid
sequenceDiagram
  participant DONOR as Donor wallet
  participant CONTRACT as Escrow contract
  participant LISTENER as EVM listener
  participant PS as Pub/Sub
  participant SQL as Cloud SQL
  participant BQ as BigQuery

  DONOR->>CONTRACT: signed contribution tx
  CONTRACT-->>LISTENER: ContributionReceived event
  LISTENER->>LISTENER: confirm chain ID / contract / block
  LISTENER->>PS: contribution.confirmed
  PS->>SQL: record contribution (idempotent)
  SQL->>BQ: normalized event
```

### Use Case 4: Milestone evaluation

Once a campaign is underway, the organization submits evidence that a funded milestone
was actually met. The document pipeline preserves the original submission while
producing a sanitized, structured version for agent consumption. The **evidence & impact
agent** retrieves the approved acceptance criteria for that milestone and compares the
submitted evidence against the criteria, the campaign's budget, invoices, and the
verified on-chain contribution — producing a sourced recommendation rather than an
unexplained score. The backend turns that recommendation into a concrete disbursement
proposal, which a **human reviewer** then evaluates in the reviewer console.

```mermaid
sequenceDiagram
  participant ORG as Organization
  participant GCS as Cloud Storage
  participant DLP as Sensitive Data Protection
  participant DOC as Document AI
  participant EVIDENCE as Evidence agent
  participant MCP as StableImpact MCP
  participant API as Domain backend
  participant REVIEWER as Human reviewer

  ORG->>GCS: upload milestone evidence
  GCS->>DLP: sanitize
  DLP->>DOC: extract fields
  DOC-->>EVIDENCE: sanitized structured evidence
  EVIDENCE->>MCP: evidence_get_extracted_fields, chain_get_transaction
  MCP-->>EVIDENCE: extracted data + verified tx
  EVIDENCE->>MCP: milestone_save_agent_review
  EVIDENCE-->>API: sourced recommendation
  API-->>REVIEWER: disbursement proposal for review
```

### Use Case 5: Disbursement

This use case is the deterministic execution of a decision a human has already made —
no agent participates in it. Once the **approval service** records the reviewer's
decision, an event activates **Workflows**, which independently re-validates that the
approval hasn't been used before, and that the milestone, amount, recipient, token, and
contract all match what was actually approved. Only after that validation and a
transaction simulation does the **human execution wallet** sign the exact approved call;
the contract's release event is then picked up by the listener and reconciled into both
the operational database and the analytics warehouse. A milestone can never be released
twice, and the released amount must exactly match what was approved.

```mermaid
sequenceDiagram
  participant REVIEWER as Human reviewer
  participant APPROVAL as Approval service
  participant PS as Pub/Sub / Eventarc
  participant WF as Workflows
  participant API as Backend - simulation
  participant WALLET as Human execution wallet
  participant CONTRACT as Escrow contract
  participant LISTENER as EVM listener
  participant SQL as Cloud SQL
  participant BQ as BigQuery

  REVIEWER->>APPROVAL: approve disbursement
  APPROVAL->>SQL: persist decision
  APPROVAL->>PS: disbursement.approved
  PS->>WF: trigger
  WF->>WF: validate approval unused, milestone, amount, recipient, token, contract
  WF->>API: simulate transaction
  API-->>WF: simulation OK
  WF->>WALLET: request signature for exact approved call
  WALLET->>CONTRACT: signed release tx
  CONTRACT-->>LISTENER: FundsReleased event
  LISTENER->>SQL: record tx hash/receipt
  SQL->>BQ: reconcile
```

### Use Case 6: Reconciliation

Independent of any single campaign action, the **audit agent** periodically (or on
request) cross-checks the platform's own claims against reality: it compares the
internal ledger against contract events, approvals, contributions, releases, and
refunds, looking for discrepancies rather than assuming the ledger is correct by
construction. Its findings, along with their sources, are published as a dataset in
BigQuery, which feeds the audit and impact dashboard that auditors and operators use.

```mermaid
sequenceDiagram
  participant AUDIT as Audit agent
  participant MCP as StableImpact MCP
  participant SQL as Cloud SQL
  participant RPC as Authorized RPC - read-only
  participant BQ as BigQuery
  participant DASH as Audit/impact dashboard

  AUDIT->>MCP: audit_get_campaign_ledger, audit_compare_sources
  MCP->>SQL: read ledger, approvals, contributions, disbursements
  MCP->>RPC: read contract events (read-only)
  AUDIT->>MCP: audit_generate_dataset, audit_save_report
  MCP->>BQ: publish audit dataset
  BQ-->>DASH: reconciliation & impact view
```

## Data Model

```mermaid
erDiagram
  ORGANIZATION ||--o{ CAMPAIGN : submits
  CAMPAIGN ||--o{ MILESTONE : defines
  MILESTONE ||--o{ EVIDENCE : receives
  EVIDENCE ||--o{ DOCUMENT_EXTRACTION : produces
  ORGANIZATION ||--o{ RISK_ASSESSMENT : assessed_by
  CAMPAIGN ||--o{ CONTRIBUTION : receives
  MILESTONE ||--o| DISBURSEMENT_PROPOSAL : generates
  DISBURSEMENT_PROPOSAL ||--|| APPROVAL : requires
  APPROVAL ||--o| DISBURSEMENT : authorizes
  CONTRIBUTION ||--|| BLOCKCHAIN_EVENT : confirmed_by
  DISBURSEMENT ||--|| BLOCKCHAIN_EVENT : confirmed_by
  CAMPAIGN ||--o{ IMPACT_METRIC : tracks
  CAMPAIGN ||--o{ AUDIT_EVENT : reconciled_via
  CAMPAIGN ||--o{ AGENT_RUN : analyzed_by
  ORGANIZATION ||--o{ INTEGRATION_EVENT : verified_via
```

Core entities, per the roadmap's minimum data model: `User`, `Organization`,
`FundingProgram`, `Campaign`, `Milestone`, `Evidence`, `DocumentExtraction`,
`RiskAssessment`, `Contribution`, `Approval`, `DisbursementProposal`, `Disbursement`,
`BlockchainEvent`, `ImpactMetric`, `AuditEvent`, `AgentRun`, `IntegrationEvent`.

Rules that apply across all of these:

- Financial records (`Contribution`, `Disbursement`) use immutable identifiers and are
  append-only — they are never mutated in place, only added to.
- Token amounts are stored as integer base units, never floating point, to avoid
  rounding drift against the on-chain amounts they must match exactly.
- `Approval` is single-use: once consumed by a `Disbursement`, it cannot authorize
  another one — this is what Workflows checks before signing.
- The backend validates every state transition; no table is written to directly by an
  agent or an external caller without passing through backend validation.
- `BlockchainEvent` always carries chain ID, token, contract address, block, and
  transaction hash, so it can be independently re-verified against the chain.
- `AgentRun` preserves the model ID, prompt version, sources cited, and trace ID for
  every agent recommendation, so a recommendation can be explained after the fact.
- `Evidence`/`DocumentExtraction` preserve hashes of both the original and sanitized
  versions of a document, so redaction can be audited without re-exposing the original.

## Security / Threat Considerations

**Identity and access.** Each component (MCP, backend, approval service, listener,
signer adapter) runs under its own least-privilege service identity; Cloud Run services
are private by default. Deployment access and runtime access are kept separate, and any
IAM change requires human approval — there is no path by which an agent or an automated
process can grant itself broader permissions.

**Agent and MCP restrictions.** StableImpact MCP is the only authoritative interface for
business operations; agents never call enterprise systems directly with shared
credentials. The MCP server enforces strict JSON Schema validation and rejects unknown
fields, authorizes every call by tool, identity, and resource, and requires idempotency
keys on writes. Critically, no MCP tool exposes generic transaction signing, token
transfer, contract deployment, or a generic write method — `disbursement_execute_approved`
is the only financial-execution tool, and it is only reachable after a valid human
approval exists; agents themselves have no private-key access at any point. Third-party
blockchain MCP tools (e.g. Alchemy) are restricted to an explicit read-only allowlist.

**AI and document security.** Model Armor inspects prompts, responses, and (where
supported) intermediate payloads for prompt injection, jailbreak attempts, and malicious
URLs, backstopped by application-level callbacks where platform coverage is incomplete.
Uploaded documents are quarantined, type- and size-restricted, and scanned before
Sensitive Data Protection redacts them — agents only ever see the sanitized, structured
output of that pipeline, never the raw upload.

**Secrets.** RPC endpoints, OAuth credentials, and provider credentials live in Secret
Manager; no secret is ever versioned into the repository, logs, prompts, traces, or
evaluation datasets, and sensitive MCP URLs/OAuth parameters are redacted wherever they
would otherwise appear in output.

**Wallet and signing separation.** At least two testnet wallets are required — a
deployment wallet (deploys the contract, assigns initial roles) and a separate execution
wallet (signs only approved releases). No wallet holds real assets, no seed phrase or
private key ever enters the repository, prompts, logs, database, analytics, or agent
memory, and the reviewer's identity is treated as distinct evidence from the execution
wallet's signature even when the same person controls both.

**Smart contract invariants.** A milestone cannot be released twice; released plus
refunded amounts can never exceed deposits; the released amount must exactly match the
approved milestone amount; only the `DISBURSEMENT_EXECUTOR_ROLE` may execute a release;
a paused campaign cannot release funds; a failed token transfer cannot leave behind
successful state; every event carries the immutable identifiers reconciliation depends
on.

**Application-level controls.** Restrictive CORS, correct token audience validation,
replay protection, idempotency on writes, and append-only audit records apply
throughout; public-facing endpoints sit behind Cloud Armor.

**Production/regulatory boundary.** Everything above is scoped to a demonstration using
synthetic organizations, documents, wallets, and testnet assets. The roadmap explicitly
does not claim authorization to process real funds — a real-funds launch would require
defined launch jurisdictions, legal review (crowdfunding, stablecoins, money
transmission, consumer protection, tax, privacy), production KYC/KYB and sanctions
screening, a custody/segregation-of-funds policy, dispute/refund/recovery processes, an
independent assessment of the contract and signing service, incident response and
continuity planning, and explicit data retention/residency decisions. None of that is
in scope for the current design.

## Deployment (Infrastructure as Code)

All cloud infrastructure is defined in Terraform and deployed through a reviewed,
repeatable pipeline rather than created by hand:

- **Provisioning.** Terraform defines every cloud resource — Cloud Run services, Cloud
  SQL instance, Cloud Storage buckets, Pub/Sub topics, Workflows definitions, BigQuery
  datasets, Secret Manager entries (created empty, populated out-of-band), and the
  per-component service accounts and their least-privilege IAM bindings. No cloud
  resource, IAM change, or API enablement is applied without an explicit human approval
  step first.
- **Application services** (StableImpact MCP, domain backend, approval service, EVM
  listener, controlled signer adapter) are each deployed as their own Cloud Run service,
  built and pushed through **Cloud Build** into **Artifact Registry**, so every deployed
  artifact is reproducible from source rather than hand-built.
- **Agent deployment** goes through the official `agents-cli scaffold` tooling into
  **Agent Runtime**, then is registered with **Gemini Enterprise** — this path is kept
  separate from the Terraform-managed application infrastructure, since it is governed
  by the Agent Platform's own deployment tooling.
- **Environments.** The roadmap targets a single demonstration environment against the
  Base Sepolia testnet; it does not define separate staging/production environments or
  a promotion pipeline between them — see Open Questions.
- **Deployment sequencing** follows the roadmap's phased plan: cloud foundation
  (Terraform, IAM, empty secrets) before agent scaffolding, before domain data, before
  the blockchain/signing path, before the MCP server and agents themselves, with human
  approval gates before any resource creation, contract deployment, or transaction.

## Assumptions and Constraints

- **Test users are seeded directly in the database.** For the current scope, beneficiary
  organization and other user accounts are created directly in Cloud SQL rather than
  through any self-service signup, invitation, or federated-SSO flow. This is a stand-in
  for the demo, not a decision about how production account provisioning should work
  (see Open Questions).
- **Testnet-only scope.** All blockchain activity targets Base Sepolia using official
  test USDC (or a clearly labeled, restricted-mint `MockUSDC` fallback if the faucet is
  unavailable). No wallet in the system holds real assets, and testnet USDC has no
  financial value.
- **Mock providers must be contract-compatible and labeled.** Any simulated external
  provider (KYC/KYB, sanctions, CRM, ERP) must preserve the real provider's API
  contract, produce reproducible positive/negative/ambiguous outcomes, be clearly
  labeled as a mock in both UI and documentation, and be swappable for the real
  provider without changing agent logic. A mock is never presented as a completed
  production integration.
- **No bridging/yield/upgrade machinery.** The fallback `MockUSDC` contract, if used, is
  a minimal six-decimal token with restricted minting only — no bridges, swaps, yield,
  or upgradability are added.
- **Deterministic execution boundary.** Agents may recommend and explain, but Workflows
  and the smart contract are the only components that actually move funds, and only
  after a human decision exists; this is a hard constraint on the design, not an
  implementation detail that could be relaxed later without changing the platform's
  core value proposition.
- **No real-funds authorization.** The design as a whole assumes a demonstration/
  hackathon context using synthetic data and testnet assets; it does not assume or
  imply any legal authorization to process real donor or grantee funds.

## Open Questions

- **Production user/account provisioning is undecided.** The source roadmap's onboarding
  flow begins with "the user authenticates," but never specifies how an organization or
  user account is created in the first place — self-service signup, an operator-driven
  invitation, or federated corporate SSO are all plausible and unresolved. The
  DB-seeded-test-user assumption above resolves this only for the current demo scope.
- **Tool-to-agent MCP assignment is partly inferred, not sourced.** The mappings shown
  under each agent in the Architecture section were inferred from prose descriptions of
  agent responsibilities, since the source roadmap lists all MCP tools in one flat
  catalog without assigning them to specific agents. Confidence is lowest for the
  **Matching agent**, whose responsibilities are described only in prose with no tool
  hints at all in the source material. This should be confirmed against the actual
  agent implementation rather than assumed from this document.
- **No defined promotion path beyond the demo environment.** The roadmap describes a
  single testnet demonstration environment; it does not define staging/production
  environments, a promotion pipeline, or how (or whether) this architecture changes
  before any real-funds launch, beyond the general regulatory boundary already noted
  under Security / Threat Considerations.
