# Business Specification: Program & Campaign Agent

## 1. Business Purpose

**The Problem:** Nonprofits often submit unstructured, vague applications. Human reviewers waste hours going back and forth to define budgets and milestones.
**The Agent's Job:** Act as an intake assistant. Translate a user's conversational funding request (e.g., _"We need $10k to build 5 solar wells in Kenya"_) into a structured, compliant Campaign Draft with clear objectives, defined milestones, and matched impact metrics.

## 2. Target User & Journey

- **Actor:** Beneficiary Organization User (interacting via Gemini Enterprise UI).
- **Trigger:** User prompts the Orchestrator (e.g., _"I want to apply for the Clean Water grant"_). The Orchestrator delegates the structuring to this Campaign Agent.
- **Outcome:** A formatted `Draft` campaign is saved to the database, ready for the Compliance Agent to review.

## 3. Core Business Rules (To be enforced by the Agent & Backend)

_Member 2 will encode these as deterministic backend rules; Member 3 will write agent evaluation tests to ensure the agent follows them._

1.  **Completeness Rule:** A campaign cannot be submitted for approval unless it contains:
    - Title and Description.
    - Target Impact Metrics (e.g., "People served", "Liters of water/day").
    - Total Requested Budget (in USDC).
    - At least **two (2) defined milestones**. (100% of funding is not provided upfront).
2.  **Budget Math Rule:**
    - The sum of all milestone budgets must **exactly equal** the Total Requested Budget.
    - _Risk Mitigation Constraint:_ No single milestone can request more than 50% of the total budget upfront.
3.  **Policy Alignment Rule:**
    - The agent must retrieve the current funding policy from Agent Platform Search.
    - It must ensure the project falls under approved enterprise CSR pillars (e.g., _WASH - Water, Sanitation, and Hygiene_, _Education_, _Climate Resilience_).
    - If the user asks for funding for an ineligible category (e.g., commercial real estate, crypto-trading), the agent must politely refuse and explain the policy.

## 4. Required Tooling (StableImpact MCP)

_Member 1 will expose these tools to the agent; the agent will use them to save your business state._

- `campaign_get_policy_context`: Retrieves the rules for the specific grant.
- `campaign_create_draft`: Saves the initial structured breakdown to Cloud SQL.
- `campaign_update_draft`: Modifies milestones or budgets if the user changes their mind.

## 5. Member 4's Acceptance Criteria (Pass/Fail)

_I will manually test these scenarios before we demo, and Member 3 will build automated regression tests for them._

- ✅ **Happy Path:** The user pastes a 2-paragraph description of a $10,000 water project. The agent replies with a structured table showing 3 milestones ($3k, $3k, $4k), suggests relevant impact metrics, and successfully saves the draft to the database.
- ❌ **Negative Test (Budget Violation):** The user asks for $10,000 but wants $8,000 in Milestone 1. The agent refuses, cites the "max 50% per milestone" rule, and proposes an alternative $5k/$5k split.
- ❌ **Negative Test (Policy Violation):** The user tries to submit a campaign to fund a commercial cryptocurrency mining farm. The agent reads the policy context, identifies the request as an excluded industry, refuses to draft the campaign, and does _not_ call the `campaign_create_draft` tool.
