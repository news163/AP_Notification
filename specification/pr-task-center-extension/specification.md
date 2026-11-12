# Specification: pr-task-center-extension

> **Guidelines**: Read [../guidelines.md](../guidelines.md) and [../guidelines-agent-extension.md](../guidelines-agent-extension.md) before executing ANY tasks below. Follow all constraints described there throughout execution.

## Basic Setup

- [ ] Read `product-requirements-document.md` and `intent.md` for full context
- [ ] Read the agent-extension skill's `references/hooks.md` (the hooks sub-skill) in full before any hook wiring steps
- [ ] Read the agent-extension skill's `references/execution.md` in full before invoking `create-agent-extension`
- [ ] Confirm base agent ORD ID and version: `sap.agtrefapp:agent:purchase-requisition-agent:v1` — version to be confirmed via agent discovery tool

## Step 0 — Feature Flag Check (MANDATORY)

- [ ] Run: `jl feature-flags get enable_agent_extension_n8n_hooks`
  - If result is `true` → proceed to Step 1
  - If result is `false` or cannot be retrieved → **STOP**. Do not add any `hooks` section to `extension.yaml` or any `n8n-workflow` entry to `asset.yaml`. Inform the user that n8n hook integration is disabled and skip all hook-related steps.
- [ ] Run: `jl feature-flags get enable_agent_extension_mcp_server`
  - Record the result for Step 2 (MCP tools section)

## Step 1 — Agent Discovery

- [ ] Use the agent extension `create-agent-extension` skill's agent discovery step to fetch the base agent details for `sap.agtrefapp:agent:purchase-requisition-agent:v1`
- [ ] From the discovery response, confirm:
  - The agent is extensible
  - `extensions[].params.supportedHooks` is non-empty — record each `hookId` value
  - If `supportedHooks` is empty or absent → **STOP**. Do not add any `hooks` section. Inform the user.
- [ ] Record the exact agent `version` from the discovery response (needed for `asset.yaml requires`)

## Step 2 — Prerequisite Check: n8n Workflow Must Exist First

- [ ] Confirm `assets/workflows/pr-task-center-notification-workflow/pr-task-center-notification-workflow.n8n.json` exists
- [ ] Confirm `assets/workflows/pr-task-center-notification-workflow/asset.yaml` exists
- [ ] If either file is missing, **execute `specification/workflows/pr-task-center-notification-workflow/specification.md` first**, then return here

## Step 3 — Hook Configuration (Post-Hook)

This is an AFTER (post-hook). Per the hooks sub-skill, only 2 parameters need to be confirmed for post-hooks:
- `hookType`: `AFTER`
- `timeout`: from the user input captured in `intent.md` — if not captured, default to `30` seconds (reasonable for Task Center creation)
- `onFailure`: `CONTINUE` (post-hook default — do not ask user)
- `canShortCircuit`: `true` (post-hook default — do not ask user)

- [ ] Read `assets/workflows/pr-task-center-notification-workflow/asset.yaml`
- [ ] Collect all `provides.apis[]` entries where `kind: rest` — each becomes one hook entry
- [ ] For each REST API entry, find the matching webhook node in the workflow JSON (by normalized path) and extract its `httpMethod`
- [ ] Record: `ordId`, `method`, workflow `name` (from `workflow.name` in `asset.yaml`)

## Step 4 — Create Asset Folder and Files

- [ ] Create folder: `assets/pr-task-center-extension/`
- [ ] Invoke `setup-solution` skill to scaffold `asset.yaml` for the agent-extension asset

### `assets/pr-task-center-extension/asset.yaml`

Must include:
- `apiVersion: asset.sap/v1`
- `kind: Asset`
- `type: agent-extension`
- `metadata.name: pr-task-center-extension`
- `provides.extensions[].descriptorFile: extension.yaml`
- `provides.extensions[].name: pr-task-center-extension`
- `requires`:
  - Agent entry: `kind: agent`, `ordId: sap.agtrefapp:agent:purchase-requisition-agent:v1`, `version: <from discovery>`
  - n8n workflow entry (if feature flag is `true`): `kind: n8n-workflow`, `name: <workflow.name from workflow asset.yaml>`

### `assets/pr-task-center-extension/extension.yaml`

- [ ] Invoke the `create-agent-extension` skill to generate `extension.yaml`
- [ ] The extension must declare only capabilities the base agent supports and whose feature flag is enabled:
  - **Instructions**: include an instruction clarifying the post-hook behaviour (optional — no mandatory instruction change for this extension)
  - **Tools**: only if `enable_agent_extension_mcp_server = true` — this extension has no MCP tools, so omit the `tools` section
  - **Hooks**: only if `enable_agent_extension_n8n_hooks = true`

### Hook entry in `extension.yaml` (one entry per REST API in the workflow)

```yaml
capabilityImplementations:
  hooks:
    - hookId: "<from base agent supportedHooks>"
      name: "PR Task Center Notification"
      hookType: AFTER
      deploymentType: N8N
      onFailure: CONTINUE
      timeout: 30
      canShortCircuit: true
      n8nWorkflowConfig:
        ordId: "<ordId of the REST API from workflow provides.apis[] where kind: rest>"
        name: "<workflow.name from workflow asset.yaml>"
        method: "<httpMethod of the matching webhook node>"
```

- [ ] Verify `n8nWorkflowConfig.ordId` is the `kind: rest` ORD ID from the workflow's `asset.yaml` — NOT the `kind: mcp-server` entry
- [ ] Verify `n8nWorkflowConfig.name` matches `workflow.name` in the workflow's `asset.yaml` exactly

## Step 5 — Update `solution.yaml`

- [ ] Add `- ref: ./assets/pr-task-center-extension/asset.yaml` to `solution.yaml` (if not already present)
- [ ] Confirm `solution.yaml` also contains `- ref: ./assets/workflows/pr-task-center-notification-workflow/asset.yaml`

## Final Validation

- [ ] `assets/pr-task-center-extension/extension.yaml` is well-formed YAML
- [ ] `assets/pr-task-center-extension/asset.yaml` is well-formed YAML
- [ ] `asset.yaml` `requires` includes:
  - `kind: agent` entry with `ordId: sap.agtrefapp:agent:purchase-requisition-agent:v1`
  - `kind: n8n-workflow` entry whose `name` matches `workflow.name` in the workflow's `asset.yaml`
- [ ] `extension.yaml` `hooks[].n8nWorkflowConfig.ordId` matches a `provides.apis[]` entry where `kind: rest` in the workflow's `asset.yaml`
- [ ] `extension.yaml` `hooks[].n8nWorkflowConfig.name` matches `workflow.name` in the workflow's `asset.yaml`
- [ ] `solution.yaml` lists both:
  - `./assets/workflows/pr-task-center-notification-workflow/asset.yaml`
  - `./assets/pr-task-center-extension/asset.yaml`
- [ ] No invented capabilities declared — only what `supportedHooks` from the base agent discovery confirms
