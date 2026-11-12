# Specification: pr-task-center-notification-workflow

> **Guidelines**: Read [../guidelines.md](../guidelines.md) and [../guidelines-n8n-workflow.md](../guidelines-n8n-workflow.md) before executing ANY tasks below. Follow all constraints described there throughout execution.

## Basic Setup

- [ ] Read `product-requirements-document.md` and `intent.md` for full context
- [ ] Read `references/workflow-hooks.md` from the n8n-workflow skill — this is a **post-hook** workflow and must follow the A2A message protocol defined there
- [ ] Read `references/n8n-sap-task-center/n8n-sap-task-center.md` from the n8n-workflow skill before generating any Task Center node
- [ ] Create asset folder: `assets/workflows/pr-task-center-notification-workflow/`

## Node Catalog Lookup (MANDATORY — do before planning any node)

- [ ] Call `search-nodes-catalog` with keywords: `["webhook", "task center", "code", "respond"]`
- [ ] Record the exact `type` and `typeVersion` for each node from the catalog results
- [ ] Confirm `CUSTOM.sapTaskCenter`, `n8n-nodes-base.webhook`, `n8n-nodes-base.code`, `n8n-nodes-base.respondToWebhook` are all available
- [ ] Do NOT use any node not returned by the catalog

## Workflow Design — Post-Hook A2A Protocol

This workflow is a **side-effect-only post-hook**. It must:
- Be triggered by a `POST` webhook (the agent runtime calls it after a PR is successfully created)
- Set `responseMode: "responseNode"` on the webhook node (mandatory for all hook workflows)
- Always return a valid A2A message with `messageId`, `role: "agent"`, and `parts: []`
- Never modify PR data — read only from the incoming A2A message

### Incoming A2A Message Structure

The A2A message body carries the agent's PR creation response in `body.parts`. Extract:
- PR number from `$input.first().json.body.parts[N].text` (or the data part if structured)
- Requester name / email
- Line items summary
- Total value
- Approving manager email (for Task Center recipient) — bind dynamically from the A2A message body

> If any field is absent from the A2A message, create the task with available fields and mark missing ones as "not provided".

## Workflow Implementation

- [ ] **Webhook trigger node**:
  - `httpMethod`: `POST`
  - `responseMode`: `"responseNode"` (MANDATORY for hook workflows)
  - Node `name` must be descriptive: `"PR Creation Post-Hook"` — used to derive the webhook path and ORD ID
  - Generate a fresh UUID for `webhookId`

- [ ] **Code node — Extract PR Details**:
  - Read the incoming A2A message: `$input.first().json.body`
  - Extract `messageId`, and PR context fields (PR number, requester, line items, total value, approving manager email) from `body.parts`
  - Return a flat object with all extracted fields so downstream nodes can reference them cleanly
  - Name: `"Extract PR Details"`

- [ ] **SAP Task Center node — `CUSTOM.sapTaskCenter`**:
  - Node `name`: `"Create PR Approval Task"` (contextual, not generic)
  - `subject`: dynamic — e.g. `"Purchase Requisition Approval Required: PR {{ $('Extract PR Details').item.json.prNumber }}"`
  - `priority`: `"MEDIUM"` (default)
  - `taskDefinition.definitionName`: `"Create PR Approval Task"`
  - **Recipient**: dynamically bind to the approving manager's email from the A2A message: `{ "userId": "={{ $('Extract PR Details').item.json.approvingManagerEmail }}" }` — do NOT hard-code a placeholder email
  - **Custom attributes** (to surface PR context in the task):
    - PR Number (`code: "prNumber"`, `type: STRING`)
    - Requester (`code: "requester"`, `type: STRING`)
    - Total Value (`code: "totalValue"`, `type: STRING`)
    - Line Items Summary (`code: "lineItemsSummary"`, `type: STRING`)
  - `onError`: `"continueRegularOutput"` — per PRD error-handling decision (continue silently on failure, never block the agent)
  - Do NOT add a `pinData` entry for the Task Center node (per Task Center rule 9)
  - **Because `onError` is `continueRegularOutput` (NOT `continueErrorOutput`)**: a single output path exits the node; NO Switch node is needed after it. The mandatory Switch-after-Task-Center pattern applies only when the Task Center is used as a human-in-the-loop gate — here it is fire-and-forget task creation. Connect the single output directly to the A2A response builder.

- [ ] **Code node — Build A2A Response**:
  - Read `messageId` from `$('PR Creation Post-Hook').first().json.body.messageId` (fallback: `'post-hook-' + Date.now()`)
  - Return a valid A2A message: `{ messageId, role: "agent", parts: [] }` — side-effect-only hook, empty parts
  - Name: `"Build A2A Response"`

- [ ] **Respond to Webhook node**:
  - Name: `"Respond to Webhook"`
  - Connect to the output of `"Build A2A Response"`
  - This node must be reachable from every execution path (including the Task Center failure path, which via `continueRegularOutput` passes input through and continues on the same output)

## Connections (topology)

```
PR Creation Post-Hook (Webhook)
  → Extract PR Details (Code)
  → Create PR Approval Task (CUSTOM.sapTaskCenter, onError: continueRegularOutput)
  → Build A2A Response (Code)
  → Respond to Webhook
```

Every execution path reaches `Respond to Webhook` — including when Task Center fails (the `continueRegularOutput` setting passes input through on the normal output).

## Asset YAML Generation

- [ ] After writing the workflow JSON, run the `generate-workflow-asset.js` script:
  ```bash
  node .agent/skills/n8n-workflow/scripts/generate-workflow-asset.js \
    assets/workflows/pr-task-center-notification-workflow/pr-task-center-notification-workflow.n8n.json \
    pr-task-center-notification-workflow \
    <solution-name>
  ```
  (Read the solution name from `solution.yaml` before running)
- [ ] Paste the generated `workflow:` and `provides.apis[]` blocks verbatim into `assets/workflows/pr-task-center-notification-workflow/asset.yaml`
- [ ] Set the `description` for the `kind: rest` entry: `"Triggered by the agent runtime after a purchase requisition is successfully created; creates a SAP Task Center approval task."`
- [ ] Verify the webhook node's `parameters.path` was rewritten to the full `<solution-name>/<workflow-name>/<operation>` path by the script

## Refine `workflow-schema.openapi.json`

- [ ] Open the generated `workflow-schema.openapi.json` sidecar
- [ ] Replace all `<...>` placeholder values (summary, response description)
- [ ] The request body is the A2A message: refine types — `messageId` (string), `role` (string, enum: user/agent), `parts` (array of objects with `kind` (string) and `text` (string, optional)), `taskId` (string), `contextId` (string)
- [ ] Confirm no `<...>` placeholders remain
- [ ] Write the refined sidecar back

## PinData

- [ ] Call `pin-data-schemas` with all node types used in the workflow (do NOT include `CUSTOM.sapTaskCenter` — per Task Center rule 9, never pin that node)
- [ ] Synthesize one sample item per returned schema and add to `pinData` (keyed by node name)
- [ ] Merge into the workflow's top-level `pinData` field

## Validation

- [ ] Call `validate-n8n-workflow` MCP tool with the full workflow JSON and the `asset.yaml` content
- [ ] Fix any errors reported before writing the file
- [ ] Write all changes in a **single Write call** (Rule 15)

## MCP Server Check (MANDATORY)

- [ ] Evaluate the three conditions in `execution.md` "MANDATORY when applicable: Generate MCP server for in-solution agent consumption":
  1. Does `solution.yaml` contain an agent asset? → If yes, proceed; if no, skip Steps A–F
  2. Does this workflow have a webhook trigger? → Yes
  3. Does any agent in the solution need to invoke this workflow? → No — this is a post-hook invoked by the agent runtime, not by an agent tool call
- [ ] Since condition 3 is NOT met (the agent runtime, not an agent asset, invokes this hook), **skip Steps A–F entirely**

## Final Checks

- [ ] `assets/workflows/pr-task-center-notification-workflow/pr-task-center-notification-workflow.n8n.json` exists with `.n8n.json` extension
- [ ] `assets/workflows/pr-task-center-notification-workflow/asset.yaml` exists with `metadata.type: n8nworkflow`
- [ ] `sourceRoot` in `asset.yaml` is `"."` (not `workflows` or any other path)
- [ ] Workflow JSON has a top-level `"description"` field
- [ ] `responseMode: "responseNode"` is set on the webhook trigger
- [ ] `CUSTOM.sapTaskCenter` node has `onError: "continueRegularOutput"`
- [ ] No `credentials` block in any node
- [ ] All `connections` reference nodes by `name`, not `id`
- [ ] `solution.yaml` includes `- ref: ./assets/workflows/pr-task-center-notification-workflow/asset.yaml`
