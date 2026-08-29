Based on the **StableImpact Team Workstream and Phase Ownership** document, as **Member 4** (Product, Business Case, User Experience, Acceptance, and Demo Owner), your primary role is to define the business logic, user experience, and success criteria for the project.

Because you share a workstream with Member 3 and generate the requirements that Members 1 and 2 build upon, you owe specific deliverables and inputs to each of your teammates. Here is exactly what you owe to Members 1, 2, and 3:

### What you owe to Member 1 (Architecture, GCP, Agents & MCP)

Member 1 relies on your business definitions to finalize architectures, and they own the ultimate deployment gates. You owe them:

- **Final Security and Evaluation Report:** You and Member 3 must deliver this jointly so Member 1 can pass the "Security and evaluation gate" before they deploy (Phase 16).
- **Shared Project Contracts:** You owe them definitions like demo acceptance criteria, evaluation dataset formats, and document fixture formats so they can review and approve them as part of the architecture gate.
- **UX/Channel Requirements:** You need to request technical implementation support from them for the user channels (Gemini Enterprise UI, Portal, Reviewer Console) based on your UX requirements.

### What you owe to Member 2 (Backend, Automation & Blockchain)

Member 2 is building the deterministic backend, state machine, and smart contracts. They cannot build these rules without your input. You owe them:

- **Business Rules & Acceptance Criteria:** You must provide the business logic governing eligibility, risk, review, and approvals so they can encode it into the domain state machine.
- **Document Fixtures & Expected States:** Realistic fixtures (documents, budgets, invoices) and the expected valid/invalid state outcomes so they can build their deterministic unit and integration tests.

### What you owe to Member 3 (Data, Integration, Security & Evaluation)

You and Member 3 are tightly coupled as co-owners of Phases 1, 6, 11, 12, 13, 14, 15, and 17. Member 3 is responsible for technically implementing the requirements that _you_ define. You owe them:

- **Phase 1 (Business Spec):** The problem statement, user journeys, business rules, policies, the synthetic case, and business acceptance criteria.
- **Phase 6 (Document Pipeline):** Realistic documents (budgets, invoices, evidence), expected outcomes, and acceptance feedback on the data extracted by Document AI.
- **Phase 11 (Enterprise Integration):** The business purpose/workflow for the selected integration, the expected behaviors (positive/negative/ambiguous), and acceptance scenarios.
- **Phase 12 (User Channels):** UX requirements, UI content, labels, help text, error messages, and what the reviewer specifically needs to see before approving.
- **Phase 13 (Analytics):** The business definitions for funding/risk/impact metrics, audit questions the dashboard must answer, and presentation requirements.
- **Phase 14 (Security/Threat Model):** Business and user misuse scenarios, privacy/consent expectations, and validation that security failures are actually understandable to human reviewers/operators.
- **Phase 15 (Evaluation):** The expected "safe" and "unsafe" agent responses to help them build regression tests, manual acceptance tests for the primary user journeys, and your review labeling incorrect agent behavior.
- **Phase 17 (Demo):** The final demo story, campaign content, submission material, presentation narrative, and the prepared list of expected judge questions/answers.
