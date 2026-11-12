# Product Requirements Document (PRD)

**Title:** Purchase Requisition Agent – Post-Hook Task Center Extension
**Date:** 2026-11-12
**Owner:** Procurement Operations
**Solution Category:** Agent Extension, n8n Workflow

---

## Product Purpose & Value Proposition

**Elevator Pitch:**
Every time the Purchase Requisition Agent creates a PR, a SAP Task Center approval task is automatically created — so no PR ever falls through the cracks and approvals are always tracked and auditable.

**Business Need:**
The Purchase Requisition Agent creates PRs on behalf of requesters, but the approval routing step is manual. Approving managers and requesters rely on informal follow-up to know that a new PR needs review. This creates delays, missed approvals, and audit gaps. A post-hook on the agent bridges this gap by automatically creating a structured Task Center task the moment a PR is successfully created.

**Expected Value:**
- 100% of successfully created PRs have a corresponding SAP Task Center approval task
- Approval routing is automated and auditable from day one
- No additional workload on the requester or approving manager — the task is created without any manual trigger

**Product Objectives:**
1. Ensure every successful PR creation triggers a Task Center task automatically
2. Capture full PR context (number, requester, line items, total value) in the task
3. Never block or degrade the agent's response — hook failures are silent and non-disruptive

---

## Business Metrics

| Metric | Baseline | Target | Timeline | Process / Capability | Source |
|--------|----------|--------|----------|----------------------|--------|
| PRs with a corresponding Task Center approval task | — | 100% of successfully created PRs | — | Purchase Requirements Processing / Approval Tracking | user |

---

## User Profiles & Personas

### Primary Persona: Procurement Requester

A business user who submits purchase requisitions through the Purchase Requisition Agent for goods or services. After the agent confirms the PR was created, they want assurance that the approval process has been kicked off automatically — without needing to manually notify anyone or follow up.

**Goals:**
- Know that the PR approval is in motion immediately after creation
- Avoid manual follow-up with the approving manager

**Key Tasks:**
- Creates PRs via the agent
- Monitors approval progress in SAP Task Center

### Secondary Persona: Approving Manager

A manager responsible for reviewing and approving new purchase requisitions. They need timely, context-rich tasks in SAP Task Center to make informed decisions without having to search for PR details in S/4HANA.

**Goals:**
- Receive complete PR details (number, requester, line items, total) in the Task Center task
- Act on approval tasks promptly without switching between systems

**Key Tasks:**
- Reviews Task Center tasks for new PRs
- Approves or rejects PRs directly in SAP Task Center

### Other User Types

- **Procurement Administrator:** Monitors that every PR has a Task Center task; investigates missing tasks if the hook fails silently.

---

## Goals and Non-Goals

### Goals (In Scope)

- Create a post-hook n8n workflow that fires after every successful PR creation by the Purchase Requisition Agent
- Create a SAP Task Center task containing PR number, requester name, line items, and total value
- Assign the Task Center task to the approving manager and requester
- Return a valid A2A response in all cases — including when Task Center task creation fails
- Register the workflow as a post-hook in the agent extension descriptor

### Non-Goals (Out of Scope)

- Sending Teams or email notifications (not required by the user)
- Modifying PR data or influencing how the agent creates PRs
- Changing any agent core behaviour (product search, PO creation, employee lookup)
- Handling PR update or cancellation events — only PR creation is in scope
- Building a custom approval UI — SAP Task Center handles the approval experience

---

## Requirements

### Must-Have Requirements

**R1: Post-hook fires only on successful PR creation**

- **Problem to Solve:** Downstream approval tasks must not be created for failed PR attempts.
- **User Story:** As a Procurement Administrator, I need the Task Center task to be created only when a PR is confirmed as successfully created, so that the audit log is accurate and clean.
- **Acceptance Criteria:**
  - Given the agent successfully creates a PR, when the post-hook fires, then a Task Center task is created.
  - Given the agent fails to create a PR, when the hook would fire, then no Task Center task is created (the hook is not called for failed agent actions).
- **Maps to Objective:** Objective 1
- **Priority Rank:** 1

**R2: PR details extracted from the A2A message**

- **Problem to Solve:** The Task Center task must contain meaningful PR context so approvers can act without switching to S/4HANA.
- **User Story:** As an Approving Manager, I need the Task Center task to include the PR number, requester name, line items, and total value, so that I can review and decide without looking up the PR separately.
- **Acceptance Criteria:**
  - Given a post-hook A2A message containing the agent's PR creation response, when the workflow runs, then it extracts PR number, requester, line items, and total value from the message body.
  - Given any of these fields are absent from the A2A message, when the workflow runs, then it creates the task with the available fields and marks missing ones as "not provided".
- **Maps to Objective:** Objectives 1 and 2
- **Priority Rank:** 2

**R3: SAP Task Center task created for the approving manager and requester**

- **Problem to Solve:** Without an automated task, the approving manager has no structured trigger to review the PR.
- **User Story:** As a Procurement Requester, I need the system to automatically notify my approving manager via a Task Center task, so that I do not have to manually chase approvals.
- **Acceptance Criteria:**
  - Given PR details are extracted, when the `CUSTOM.sapTaskCenter` node runs, then a Task Center task is created and assigned to the approving manager and requester.
  - Given the task is created, then it includes PR number, requester name, line items summary, and total value in the task description.
- **Maps to Objective:** Objectives 1 and 2
- **Priority Rank:** 3

**R4: Graceful failure — Task Center errors never block the agent response**

- **Problem to Solve:** If Task Center is unavailable, the agent's response to the user must not be blocked or delayed.
- **User Story:** As a Procurement Requester, I need the agent to respond normally even if the Task Center task creation fails, so that my workflow is not interrupted.
- **Acceptance Criteria:**
  - Given the `CUSTOM.sapTaskCenter` node fails, when the workflow handles the error, then it continues on the regular output path (`continueRegularOutput`) and returns a valid A2A message with empty parts.
  - Given the Task Center failure, then no error is surfaced to the user in the agent response.
- **Maps to Objective:** Objective 3
- **Priority Rank:** 4

**R5: Valid A2A response always returned**

- **Problem to Solve:** Post-hook workflows must conform to the A2A protocol; an invalid or empty response breaks the agent pipeline.
- **User Story:** As a platform operator, I need the hook workflow to always return a valid A2A message, so that the agent pipeline continues correctly.
- **Acceptance Criteria:**
  - Given any execution path (success or failure), when the workflow ends, then the "Respond to Webhook" node returns a valid A2A message (`messageId`, `role`, `parts: []`).
- **Maps to Objective:** Objective 3
- **Priority Rank:** 5

**R6: Hook registered as AFTER hook in the agent extension**

- **Problem to Solve:** The hook must be formally declared in the extension descriptor so the agent runtime invokes it after PR creation.
- **User Story:** As a Procurement Administrator, I need the extension to be correctly wired as a post-hook so that it fires automatically without any manual configuration.
- **Acceptance Criteria:**
  - Given the extension.yaml is deployed, then it declares `hookType: AFTER` with `deploymentType: N8N` and references the workflow's REST API ORD ID.
  - Given the agent extension is active, then the hook fires after every successful agent interaction that results in a PR creation.
- **Maps to Objective:** Objective 1
- **Priority Rank:** 6

---

## Solution Architecture

**Architecture Overview:**
The solution consists of two deployed assets: an agent extension descriptor and a post-hook n8n workflow. The agent extension registers the workflow as an AFTER hook on the Purchase Requisition Agent. After the agent confirms a PR is created, the runtime calls the workflow webhook with an A2A message containing the agent's response. The workflow extracts PR details, creates a SAP Task Center task, and returns a valid A2A response.

**Key Components:**

- **Agent Extension (`extension.yaml` + `asset.yaml`):** Declares the AFTER hook targeting `sap.agtrefapp:agent:purchase-requisition-agent:v1`. Registers the n8n workflow as a dependency.
- **Post-Hook n8n Workflow (`.n8n.json`):** Webhook-triggered workflow that parses the A2A message, extracts PR details, invokes `CUSTOM.sapTaskCenter` to create an approval task, handles errors silently, and responds with a valid A2A message.
- **SAP Task Center (`CUSTOM.sapTaskCenter` node):** Creates the approval/review task and assigns it to the approving manager and requester.

**Integration Points:**

- **A2A Message (inbound):** The agent runtime POST-calls the workflow webhook with the agent's A2A response after PR creation. PR details are embedded in the message parts.
- **SAP Task Center (outbound):** The `CUSTOM.sapTaskCenter` node writes a new task. Write operation — creates a task record.

### Automation & Agent Behaviour

**Automation Level:** Rule-based (no AI inference in the hook)

**Actions the system performs without human approval:**
- Extracts PR details from the incoming A2A message
- Creates a SAP Task Center approval task with PR context

**Actions that require human review or approval:**
- The Task Center task itself represents the human approval step — the manager reviews and approves/rejects the PR in SAP Task Center

**Knowledge & data sources accessed:**
- A2A message body (inbound): PR number, requester, line items, total value — sourced from the agent's creation response

**Tools or connectors invoked:**
- `CUSTOM.sapTaskCenter`: Creates the approval task in SAP Task Center — write operation; creates a new task record
- `n8n-nodes-base.respondToWebhook`: Returns the A2A response to the agent runtime — read-only

**Guardrails & fail-safes:**
- The hook must never modify PR data — it is strictly read-only on the A2A message
- If Task Center creation fails, the workflow continues on the regular output path and returns a valid A2A message with empty parts — the agent response is never blocked
- The hook fires only when the agent runtime calls it after a successful PR creation; it has no trigger of its own for failed PRs

---

## Milestones

### M1: Hook Workflow Deployed

- **Description:** The post-hook n8n workflow is deployed, registered, and reachable via its webhook endpoint.
- **Achieved when:** The workflow asset is deployed and the webhook endpoint responds to a test POST request.
- **Log on achievement:** `M1.achieved: post-hook workflow deployed and webhook endpoint is reachable`
- **Log on miss:** `M1.missed: post-hook workflow deployment failed or webhook endpoint is not reachable`

### M2: Agent Extension Active

- **Description:** The agent extension descriptor is deployed and the AFTER hook is registered on the Purchase Requisition Agent.
- **Achieved when:** The extension.yaml is deployed and the agent runtime recognises the AFTER hook for the base agent.
- **Log on achievement:** `M2.achieved: agent extension active — AFTER hook registered on purchase-requisition-agent`
- **Log on miss:** `M2.missed: agent extension deployment failed or hook not recognised by the agent runtime`

### M3: Task Center Task Created on PR Creation

- **Description:** End-to-end flow is verified: a PR creation by the agent triggers the hook and produces a Task Center task.
- **Achieved when:** A Task Center task containing the correct PR details (number, requester, line items, total) is created after a test PR creation.
- **Log on achievement:** `M3.achieved: Task Center task created successfully for PR {pr_number}`
- **Log on miss:** `M3.missed: Task Center task was not created after PR creation — check CUSTOM.sapTaskCenter node and task assignment configuration`

---

## Risks, Assumptions, and Dependencies

### Risks

- **Task Center configuration:** If the Task Center instance is not properly configured with the correct recipients, tasks may be created but not reach the right approvers.
- **A2A message structure:** If the Purchase Requisition Agent changes the format of its PR creation response (A2A message parts), the hook's extraction logic may need updating.

### Assumptions

- The Purchase Requisition Agent's A2A response includes PR number, requester name, line items, and total value in the message parts after a successful creation.
- The `CUSTOM.sapTaskCenter` node has access to the required Task Center service credentials.
- The agent extension framework supports `hookType: AFTER` for the target base agent (`sap.agtrefapp:agent:purchase-requisition-agent:v1`).
- The `enable_agent_extension_n8n_hooks` feature flag is enabled in the environment.

### Dependencies

- SAP Task Center service must be active and reachable from the n8n instance
- The base agent (`purchase-requisition-agent:v1`) must expose at least one supported hook in `extensions[].params.supportedHooks`
- The n8n instance must have the `CUSTOM.sapTaskCenter` node available (confirmed)

---

## Appendix

### References

- SAP Task Center documentation: https://help.sap.com/docs/task-center
- Purchase Requisition API (S/4HANA): `sap.s4:apiResource:CE_PURCHASEREQUISITION_0001:v1`
- Base agent ORD ID: `sap.agtrefapp:agent:purchase-requisition-agent:v1`
- A2A message protocol: Agent-to-Agent hook message format (see workflow-hooks reference)
