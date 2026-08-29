# StableImpact Team Workstream and Phase Ownership

## 1. Purpose

This document assigns the StableImpact execution roadmap across four team members. Each person is accountable for the outcome of their assigned phases, coordinates dependencies with the other workstreams, and preserves evidence in the shared project documentation.

Phase ownership does not mean working in isolation. Member 1 remains responsible for architecture and integration quality, while each workstream owner drives execution, testing, documentation, and handoff for their assigned scope.

---

## 2. Ownership summary

| Member | Assigned phases |
|---|---|
| **Member 1** | **0, 2, 3, 4, 9, 10 and 16** |
| **Member 2** | **5, 7 and 8** |
| **Member 3** | **Shared ownership of 1, 6, 11, 12, 13, 14, 15 and 17** |
| **Member 4** | **Shared ownership of 1, 6, 11, 12, 13, 14, 15 and 17** |

---

## 3. Member 1

### Responsibility area

**Google Cloud, Gemini Agents and MCP**

### Assigned phases

- Phase 0 — Resource Discovery
- Phase 2 — Official Patterns
- Phase 3 — Cloud Foundation
- Phase 4 — Agent Scaffold
- Phase 9 — MCP Server
- Phase 10 — ADK Agents
- Phase 16 — Deployment and Publication

### Primary responsibilities

- Own the end-to-end technical architecture.
- Validate the Google Cloud project, quotas, APIs, regions, identities, and hackathon resources.
- Define service boundaries, interfaces, schemas, events, and authentication patterns.
- Establish the Google Cloud foundation with Terraform, IAM, Artifact Registry, Cloud Build, Secret Manager, and required services.
- Create and preserve the official `agents-cli` project scaffold.
- Implement the ADK orchestrator and specialist-agent topology.
- Implement and secure StableImpact MCP on Cloud Run.
- Configure Gemini Enterprise, Agent Runtime, Agent Registry, Agent Gateway, Model Armor, and related services when available.
- Review integration points across backend, blockchain, document processing, enterprise systems, analytics, and user channels.
- Own the final deployment and Gemini Enterprise publication process.
- Ensure that no agent can approve, sign, or independently execute a financial transaction.

### Phase deliverables

#### Phase 0 — Resource Discovery

- `status/RESOURCES.md` with APIs, quotas, regions, identities, permissions, wallets, testnet resources, and provider access.
- Availability and fallback classification for every planned component.
- Confirmed Base Sepolia parameters and approved RPC strategy.
- Documented MCP provider permissions.

#### Phase 2 — Official Patterns

- Reviewed official Google ADK and Gemini Enterprise samples.
- Recorded architecture decisions in `status/DECISIONS.md`.
- Selected patterns for MCP, authentication, delegated identity, safety, search, evaluation, deployment, and publication.

#### Phase 3 — Cloud Foundation

- Terraform structure.
- Separated service accounts.
- Least-privilege IAM.
- Artifact Registry and Cloud Build configuration.
- Storage, database, analytics, and event resource definitions.
- Secret Manager entries without versioned values.

#### Phase 4 — Agent Scaffold

- Valid `agents-cli` project.
- Preserved evaluation, CI, deployment, and observability structure.
- Base agent running locally.
- Approved dependency and model configuration.

#### Phase 9 — MCP Server

- Private Streamable HTTP MCP server on Cloud Run.
- Strict tool schemas.
- Identity- and resource-level authorization.
- Audit logging and sensitive-data redaction.
- External blockchain MCP allowlist.
- Negative tests proving that generic signing, transfer, deployment, and contract-write tools are unavailable to agents.

#### Phase 10 — ADK Agents

- Orchestrator and specialist agents.
- Minimum tool allocation per agent.
- Search, session, safety, trace, and audit callbacks.
- Grounded responses and financial-action refusal behavior.
- Local agent smoke tests.

#### Phase 16 — Deployment and Publication

- Approved deployment evidence.
- Agent Runtime deployment.
- Cloud Run deployment for MCP and coordinated application services.
- Deployed smoke-test results.
- Gemini Enterprise registration and access verification.
- Final environment and permission review.

### Required collaboration

- Provide interface contracts to all team members before dependent implementation.
- Review database, contract, Workflow, and event designs owned by Member 2.
- Support technical implementation requested by Members 3 and 4.
- Receive the security and evaluation gate from Member 3 before deployment.

---

## 4. Member 2

### Responsibility area

**Test Automation, Domain Reliability, Backend and Blockchain Engineer**

### Assigned phases

- Phase 5 — Domain Data and State Machine
- Phase 7 — Blockchain and Controlled Signing
- Phase 8 — Backend and Workflows

### Primary responsibilities

- Turn business rules into deterministic automated tests.
- Implement and validate the domain state machine.
- Own automated protection against reused approvals, duplicate events, and duplicate disbursements.
- Implement the Base Sepolia contract and contract test suite.
- Implement RPC access, blockchain adapters, and the EVM event listener.
- Implement backend services and deterministic Workflows.
- Automate integration tests across Cloud SQL, backend, Workflows, RPC, contract events, and reconciliation.
- Validate failure handling, retries, idempotency, and transaction simulation.

### Phase deliverables

#### Phase 5 — Domain Data and State Machine

- Cloud SQL schema and migrations.
- Domain entities and state-transition rules.
- Append-only financial ledger.
- Idempotency and uniqueness controls.
- Unit and integration tests for invalid states.
- Automated tests for reused approvals and duplicate releases.

#### Phase 7 — Blockchain and Controlled Signing

- Base Sepolia contract using official test USDC or clearly labeled `MockUSDC`.
- Contract tests covering roles, deposits, milestone releases, refunds, pause, reentrancy, transfer failures, and double release.
- Dedicated RPC adapter.
- EVM event listener for Pub/Sub ingestion.
- Deployment scripts that never expose private keys.
- Explorer verification evidence.
- Reproducible contribution and disbursement transaction tests.
- Proof that the agent cannot access deployment or execution keys.

#### Phase 8 — Backend and Workflows

- Campaign, evidence, approval, ledger, and audit APIs.
- Approval service.
- Deterministic disbursement Workflow.
- Transaction simulation path.
- Controlled execution integration.
- Pub/Sub and Eventarc integration.
- Automated tests for unavailable RPC, failed simulation, invalid approval, wrong contract, wrong chain, signer rejection, and duplicate retry.

### Quality gate

This workstream is complete only when:

- Invalid state transitions are rejected.
- A disbursement cannot execute without a valid single-use approval.
- Retries do not create duplicate transactions.
- Contract and backend state can be reconciled.
- Test results are reproducible by another team member.

### Required collaboration

- Use schemas and security boundaries provided by Member 1.
- Receive business rules and acceptance criteria from Member 4.
- Provide normalized events and test fixtures to Member 3.
- Escalate architectural or signing-policy decisions to Member 1.

---

## 5. Shared Workstream — Members 3 and 4

### Shared role

**Business, Data, Evidence, Enterprise Integration, User Experience, Security, Agent Quality and Demonstration**

### Shared assigned phases

- Phase 1 — Business Specification
- Phase 6 — Document Pipeline
- Phase 11 — Enterprise Integration
- Phase 12 — User Channels
- Phase 13 — Analytics and Observability
- Phase 14 — Security and Threat Model
- Phase 15 — Tests and Agent Evaluation
- Phase 17 — Demonstration Evidence

Both members are accountable for the results of these phases. They divide the work according to their skills, review each other’s outputs, and deliver one joint handoff to the rest of the team.

### Internal responsibility split

| Phase | Member 3 | Member 4 |
|---|---|---|
| **1 — Business Specification** | Validates technical feasibility, data requirements, integrations, and measurable acceptance criteria | Leads the problem statement, users, business rules, policies, synthetic case, and business acceptance |
| **6 — Document Pipeline** | Implements storage, sanitization, Document AI, events, hashes, and automated document tests | Creates realistic documents and expected outcomes; validates that extracted information is understandable and useful |
| **11 — Enterprise Integration** | Implements authentication, adapters, transformations, idempotency, and integration tests | Defines the business workflow, selected enterprise system, expected behavior, and acceptance scenarios |
| **12 — User Channels** | Supports channel integration, data flow, authentication testing, and technical validation | Leads UX requirements, content, reviewer experience, user acceptance, and accessibility of the flow |
| **13 — Analytics and Observability** | Implements BigQuery, correlation, reconciliation, traces, metrics, and alerts | Defines the business metrics, audit questions, dashboard meaning, and presentation requirements |
| **14 — Security and Threat Model** | Leads threat modeling, technical security tests, IAM review, prompt-injection testing, and remediation evidence | Defines misuse scenarios, privacy expectations, user risks, and verifies that controls are understandable |
| **15 — Tests and Agent Evaluation** | Implements evaluation datasets, pytest coverage, `agents-cli eval`, regression automation, and results | Defines expected answers, reviews agent quality, performs acceptance tests, and labels incorrect or unsafe behavior |
| **17 — Demonstration Evidence** | Produces technical evidence, metrics, traces, screenshots, fallback proof, and reproducibility material | Leads the demo story, campaign content, submission material, presentation, and judge-question preparation |

---

## 6. Member 3

### Responsibility area

**Data, Evidence, Enterprise Integration, Security and Evaluation Engineer**

### Assigned phases

Shares ownership of **Phases 1, 6, 11, 12, 13, 14, 15 and 17** with Member 4.

### Primary responsibilities

- Own the evidence-processing pipeline.
- Integrate Cloud Storage, Sensitive Data Protection, and Document AI.
- Implement or coordinate the selected enterprise integration.
- Build BigQuery datasets, views, reconciliation queries, and audit outputs.
- Define observability fields, dashboards, traces, and alerts.
- Create the threat model and security-test cases.
- Build agent evaluation datasets and regression tests.
- Validate grounding, tool choice, policy compliance, and resistance to prompt injection.
- Provide an independent quality gate before deployment.
- Convert Member 4’s requirements and acceptance scenarios into technical fixtures, tests, integrations, and measurable evidence.

### Phase deliverables

#### Phase 6 — Document Pipeline

- Signed upload flow.
- Restricted document quarantine.
- Sensitive-data inspection and redaction.
- Document AI extraction.
- Original and sanitized document hashes.
- Pub/Sub document events.
- Valid, incomplete, ambiguous, and malicious document fixtures.

#### Phase 11 — Enterprise Integration

- At least one visible enterprise integration or an approved equivalent.
- Authentication and transformation rules.
- Idempotent external writes.
- Clearly labeled contract-compatible mocks for unavailable providers.
- Evidence showing that agents use domain tools rather than generic vendor APIs.

#### Phase 13 — Analytics and Observability

- BigQuery datasets and normalized event views.
- Operational and blockchain reconciliation queries.
- Agent, approval, Workflow, and transaction correlation.
- Logging, Trace, and Monitoring configuration.
- Audit or impact view for the demonstration.
- Alerts for failures, policy rejections, and reconciliation differences.

#### Phase 14 — Security and Threat Model

- `docs/threat-model.md`.
- Prompt-injection and malicious-document tests.
- Tool-authorization tests.
- Replay, duplicate, wrong-network, and wrong-contract tests.
- IAM, secrets, storage, MCP, signer, and data-exposure review.
- Evidence that no payment path bypasses human approval.

#### Phase 15 — Tests and Agent Evaluation

- Deterministic pytest coverage where appropriate.
- `agents-cli eval` datasets and criteria.
- Tests for tool selection, grounding, security, policy compliance, and financial-action refusal.
- Regression cases for every critical failure.
- Evaluation results and limitations.

### Quality gate

Before Phase 16, this workstream must confirm:

- Zero payments without valid approval.
- Zero exposed secrets.
- Zero duplicate releases.
- All sensitive actions are auditable.
- Critical prompt-injection cases are rejected.
- Required agent-evaluation thresholds are satisfied.

### Asynchronous-work advantage

This workstream should leave completed tests, reports, and blockers before Members 1, 2, and 4 begin their next working period. Its outputs must be independently reviewable without requiring a live meeting.

### Required collaboration

- Consume normalized events and fixtures from Member 2.
- Use identity, tracing, MCP, and deployment standards from Member 1.
- Define and execute every shared phase together with Member 4.
- Convert campaign content and acceptance criteria from Member 4 into implementations and automated checks.
- Review user-channel and demo requirements for technical feasibility.
- Deliver the final security and evaluation report to Member 1.

---

## 7. Member 4

### Responsibility area

**Product, Business Case, User Experience, Acceptance and Demo Owner**

### Assigned phases

Shares ownership of **Phases 1, 6, 11, 12, 13, 14, 15 and 17** with Member 3.

### Ownership model

Member 4 is accountable for requirements, content, user experience, business acceptance, and presentation across every shared phase. Member 3 leads the related technical implementation, and Member 1 provides architecture or user-channel support when required. Member 4 decides whether the result satisfies the intended business and user experience.

### Primary responsibilities

- Own the business problem and primary use case.
- Define the synthetic nonprofit, funding program, campaign, budget, milestones, evidence, and impact indicators.
- Define internal and external users.
- Write eligibility, risk, review, and approval requirements in business language.
- Define what the reviewer must see before approving.
- Prepare UI content, labels, help text, and error messages.
- Perform acceptance testing as an operator, nonprofit user, donor, and reviewer.
- Own the hackathon submission content and demo narrative.
- Maintain the list of expected judge questions and business answers.
- Verify that mocks and testnet assets are clearly identified.
- Review document extraction, integrations, analytics, threat scenarios, and agent evaluations from a business-user perspective.

### Phase deliverables

#### Phase 1 — Business Specification

- Approved `.agents-cli-spec.md` business content.
- Primary clean-water or selected impact scenario.
- User roles and user journeys.
- Eligibility, risk, evidence, approval, and impact criteria.
- Sensitive-data and retention expectations.
- Selected enterprise integration from a business perspective.
- `docs/business-case.md`.

#### Phase 6 — Document Pipeline

- Realistic nonprofit, budget, invoice, milestone, and evidence fixtures.
- Expected values for every document.
- Acceptance feedback on sanitized and extracted results.
- Clearly documented ambiguous, incomplete, and malicious scenarios.

#### Phase 11 — Enterprise Integration

- Business purpose and workflow for the selected integration.
- Expected positive, negative, and ambiguous provider outcomes.
- Acceptance criteria proving that the integration adds visible enterprise value.
- Clear labeling of mocks and limitations.

#### Phase 12 — User Channels

- UX requirements for Gemini Enterprise, the external portal, and reviewer console.
- Screen content and acceptance criteria.
- Required evidence, risk, amount, recipient, token, contract, and network presentation.
- Authentication and permission expectations expressed as user scenarios.
- Acceptance-test results.
- Verified transaction links visible to the user.

Technical implementation is coordinated with Member 1. Member 4 is not required to write application code.

#### Phase 13 — Analytics and Observability

- Business definitions for funding, evidence, risk, reconciliation, and impact metrics.
- Audit questions that BigQuery and the dashboard must answer.
- Acceptance review of charts, tables, and operational explanations.

#### Phase 14 — Security and Threat Model

- User and business misuse scenarios.
- Privacy, consent, and sensitive-data expectations.
- Review of whether security failures and warnings are understandable to operators and reviewers.

#### Phase 15 — Tests and Agent Evaluation

- Expected safe and unsafe agent responses.
- Manual acceptance tests for the primary user journeys.
- Review and classification of agent-evaluation failures.
- Business approval of the final evaluation evidence.

#### Phase 17 — Demonstration Evidence

- Complete synthetic campaign and source material.
- Safe documents, invoices, milestone evidence, and expected outcomes.
- `docs/demo-script.md`.
- Final business narrative.
- Demo checklist and fallback narrative.
- Screenshots or evidence of agents, MCP, Workflows, BigQuery, approvals, and transactions.
- Submission text and presentation material.
- Judge-question preparation.

### Acceptance gate

The demo is not ready until this workstream confirms that:

- The problem and value are understandable without technical explanation.
- The reviewer understands exactly what is being approved.
- The distinction between agent recommendation and human decision is visible.
- Testnet assets and mocked providers are clearly labeled.
- The audit explanation answers who approved, what evidence was used, what amount was released, and where the transaction can be verified.

### Required collaboration

- Provide business rules and fixtures to Members 2 and 3.
- Work jointly with Member 3 on all eight shared phases.
- Review the technical outputs of documents, integrations, analytics, security, and evaluations.
- Request technical implementation support from Member 1 for user channels.
- Validate integrated features as a real user.
- Own the final presentation while technical members provide evidence and answer specialist questions.

---

## 8. Cross-workstream dependency flow

```text
Phase 0 — Member 1 establishes resources and constraints
    |
    +--> Phase 1 — Members 3 and 4 define the approved business specification
    |
    +--> Phase 2 — Member 1 selects official implementation patterns
             |
             v
Phase 3 — Cloud foundation
    |
    v
Phase 4 — Agent scaffold and shared interfaces
    |
    +--> Phases 5, 7, 8 — Automation, backend and blockchain
    |
    +--> Phases 6, 11 — Shared workstream delivers documents and enterprise integration
             |
             v
Phase 9 — StableImpact MCP integration
    |
    v
Phase 10 — ADK agent integration
    |
    +--> Phase 12 — Shared workstream delivers user channels and acceptance
    |
    +--> Phase 13 — Shared workstream delivers analytics and observability
             |
             v
Phase 14 — Shared security gate
    |
    v
Phase 15 — Shared evaluation gate
    |
    v
Phase 16 — Deployment and publication
    |
    v
Phase 17 — Shared final demonstration
```

No owner should change another workstream’s interface without recording the decision and notifying every affected owner.

---

## 9. Shared project contracts

The following artifacts are shared interfaces and require Member 1 review:

- MCP tool names and JSON schemas.
- Database entities and migration contracts.
- Pub/Sub event names and payload schemas.
- Workflow input and output schemas.
- Smart-contract ABI, addresses, roles, and event definitions.
- Authentication and authorization expectations.
- Correlation identifiers.
- Document fixture formats.
- Evaluation dataset formats.
- Demo acceptance criteria.

Interface changes must be recorded in `status/DECISIONS.md` and reflected in `status/TRACEABILITY.md`.

---

## 10. Time-zone coordination

Member 3 works 10 hours and 30 minutes ahead of Members 1, 2, and 4. The preferred live synchronization window is:

- Members 1, 2, and 4: 9:00–9:30 a.m. local time.
- Member 3: 7:30–8:00 p.m. local time.

Most coordination should remain asynchronous.

### Follow-the-sun workflow

1. The two shared-workstream owners agree on requirements, expected outcomes, and the next handoff.
2. Member 3 completes implementation, integration, analytics, security, or evaluation work.
3. They publish evidence, test results, blockers, and the next requested action.
4. Members 1, 2, and 4 review those outputs at the beginning of their working period.
5. They record acceptance feedback, answer blockers, and leave updated requirements before Member 3 begins again.

Avoid assigning urgent wallet signatures, IAM approvals, or production deployments to Member 3 when no approver is available.

---

## 11. Required handoff format

Every workstream update must include:

```text
Completed:
- Work that is finished and verified.

Changed:
- Files, schemas, services, contracts, or configuration modified.

Evidence:
- Tests, evaluation results, screenshots, logs, explorer links, or transaction hashes.

Needs Review:
- Decisions or outputs another owner must validate.

Blockers:
- Missing access, permissions, dependencies, decisions, or resources.

Next Action:
- One concrete next step and its owner.
```

Shared status files:

- `status/STATUS.md`
- `status/DECISIONS.md`
- `status/RISKS.md`
- `status/RESOURCES.md`
- `status/TRACEABILITY.md`

---

## 12. Review and approval gates

### Architecture gate

Owner: Member 1.

Required before dependent implementation:

- Approved service boundaries.
- Shared contracts.
- Identity and authorization design.
- Core fallback decisions.

### Transaction-reliability gate

Owner: Member 2.

Required before integrated agent testing:

- Deterministic state machine.
- Contract and backend tests.
- Idempotent Workflow behavior.
- Reproducible blockchain transactions.

### Security and evaluation gate

Owners: Members 3 and 4. Member 3 leads technical validation; Member 4 validates business safety and expected behavior.

Required before deployment:

- Threat-model review.
- Security-test results.
- Agent-evaluation thresholds.
- No critical unresolved finding.

### Product acceptance gate

Owners: Members 3 and 4. Member 4 leads business and UX acceptance; Member 3 confirms technical evidence and reproducibility.

Required before the final demonstration:

- Understandable user journey.
- Complete synthetic case.
- Reviewer acceptance.
- Clear business value.
- Complete demo evidence and fallback narrative.

### Deployment gate

Owner: Member 1, with explicit human approval.

Required before external changes:

- Passed transaction, security, evaluation, and product gates.
- Approved cloud and blockchain actions.
- Verified permissions and secrets.

---

## 13. Team Definition of Done

The team assignment is complete when:

- Every phase has clearly defined accountable owner or co-owners.
- Phases 1, 6, 11, 12, 13, 14, 15 and 17 have joint accountability between Members 3 and 4.
- Shared interfaces have Member 1 approval.
- Each phase produces evidence and satisfies its exit criteria.
- The time-zone handoff works without relying on undocumented context.
- The non-technical owner can validate the complete user experience without reading implementation code.
- Automated tests protect domain, blockchain, backend, Workflow, and agent behavior.
- Security and agent-evaluation gates pass before deployment.
- The deployment and demonstration use only approved resources and testnet assets.
- The final demo clearly connects business value, Gemini agents, MCP, Google Cloud, human approval, and verifiable blockchain evidence.
