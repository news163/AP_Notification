# Purchase Requisition Agent – Post-Hook Task Center Extension

Agent extension for the Purchase Requisition Agent: after a PR is successfully created, automatically create a SAP Task Center approval/review task so that every new PR is tracked and auditable.

## Business challenge

After the Purchase Requisition Agent creates a purchase requisition, there is no automated downstream step to route the PR for approval. Approving managers and requesters rely on manual follow-up. The goal is to add a post-hook to the agent that fires only on successful PR creation and creates a SAP Task Center task — capturing the PR number, requester, line items, and total value — so approvals are always tracked and auditable without changing the agent's core behaviour.

## Business Goals & Success Criteria

| Metric | Baseline | Target | Timeline | Process / Capability | Source |
|--------|----------|--------|----------|----------------------|--------|
| Every new PR gets a Task Center task | — | 100% of successfully created PRs have a corresponding Task Center task | — | Purchase Requirements Processing / Approval Tracking | user |

## Key Milestones

| Milestone | Condition |
|-----------|-----------|
| Hook workflow deployed | Post-hook n8n workflow is registered and reachable |
| Agent extension active | Extension.yaml is deployed and linked to the base agent |
| Task created on PR creation | A Task Center task is produced for every successful PR creation event |

## Business Architecture (RBA)

### End-to-End Process

Source to Pay (generic)

### Process Hierarchy

```
Source to Pay (E2E)
└── Procure to Receipt (generic)
    └── Purchase products and services (BPS-329)
        └── Requisition product or service
        └── Approve / review purchase requisition (post-creation task routing)
```

### Summary

The PR approval routing requirement maps to the Source to Pay E2E → Procure to Receipt → Purchase products and services sub-process (BPS-329). The hook adds a post-creation approval task step that is currently a gap in the automated agent flow.

## Fit Gap Analysis

| Requirement (business) | Standard asset(s) found | API ORD ID | MCP Server ORD ID | MCP Server Version | Webhook API ORD ID | Data Product ORD ID | Gap? | Notes / assumptions |
| ---------------------- | ----------------------- | ---------- | ----------------- | ------------------ | ------------------ | ------------------- | ---- | ------------------- |
| Read created PR details (post-hook context) | Purchase Requisition API (S/4HANA) | `sap.s4:apiResource:CE_PURCHASEREQUISITION_0001:v1` | — | — | — | — | No | PR details (number, requester, items, total) are passed via the A2A message from the agent; no direct API call needed in the hook |
| Create Task Center approval task | SAP Task Center (`CUSTOM.sapTaskCenter`) | — | — | — | — | — | No | `CUSTOM.sapTaskCenter` node is available on this n8n instance |
| Post-hook trigger (AFTER PR creation) | Agent extension post-hook mechanism | — | — | — | — | — | No | `hookType: AFTER` supported by agent extension framework |
| Additional notifications (Teams / Email) | Microsoft Teams, Outlook nodes available | — | — | — | — | — | No | User confirmed Task Center task only — no additional notification needed |
| Task Center failure resilience | n8n `continueRegularOutput` | — | — | — | — | — | No | On Task Center failure: continue silently, return valid A2A response without blocking the agent |

### Key findings

- `CUSTOM.sapTaskCenter` node is available on this n8n instance — no gap for task creation.
- The A2A post-hook message carries the agent's output (PR details) — no extra SAP API call is required in the hook to read PR data.
- No MCP servers found for the Purchase Requisition APIs; not needed since PR data is sourced from the A2A message.
- SAP Task Center direct API (`sap.afc:apiResource:TaskProviderV2:v1`) is marked unavailable — the `CUSTOM.sapTaskCenter` n8n node is the correct channel.
- User confirmed: Task Center task only, no Teams/email notification.
- On Task Center failure, the workflow continues silently — the hook must not block the agent's response.

## Recommendations

### Post-Hook n8n Workflow for PR Approval Task Creation

#### Executive Summary

Add a post-hook workflow that creates a Task Center task on every successful PR creation.

#### Recommended Solution

Build an **agent extension** targeting `sap.agtrefapp:agent:purchase-requisition-agent:v1` with a single **post-hook n8n workflow**. The workflow:
1. Is triggered via an HTTP webhook (`hookType: AFTER`) after the agent successfully creates a PR.
2. Reads PR details (number, requester, line items, total value) from the incoming A2A message.
3. Creates a SAP Task Center task for the approving manager and requester using the `CUSTOM.sapTaskCenter` node.
4. If Task Center creation fails (`continueRegularOutput`), continues silently and returns a valid A2A message.
5. Always returns a valid A2A response (`parts: []`) — it is a side-effect-only post-hook.

#### Recommended solution category

Agent Extension, n8n Workflow

### Error Handling (per SAP custom node)

| Node (business step) | On error | Error-path action |
|---|---|---|
| Create Task Center task (`CUSTOM.sapTaskCenter`) | Continue with last data (`continueRegularOutput`) | Return minimal valid A2A message silently |

#### Intent fit
95%
