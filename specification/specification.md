# Specification

> **Guidelines**: Read [guidelines.md](./guidelines.md) before executing ANY tasks below.

This solution consists of two assets:
1. **`pr-task-center-notification-workflow`** — post-hook n8n workflow that creates a SAP Task Center task after a PR is successfully created
2. **`pr-task-center-extension`** — agent extension that registers the workflow as an AFTER hook on the Purchase Requisition Agent

**Important execution order**: the n8n workflow asset MUST be built before the agent extension asset, because the extension wires to the workflow's deployed REST API ORD ID and workflow name.

Check off items as completed.

## Solution Setup

- [x] Invoke `setup-solution` skill to create `solution.yaml` and scaffold asset folders:
  - `assets/workflows/pr-task-center-notification-workflow/`
  - `assets/pr-task-center-extension/`
- [x] Validate `solution.yaml` is well-formed and lists both asset refs once created

## Asset 1 — n8n Workflow (build first)

- [x] Execute `specification/workflows/pr-task-center-notification-workflow/specification.md` (all items)

## Asset 2 — Agent Extension (build after workflow is complete)

- [x] Execute `specification/pr-task-center-extension/specification.md` (all items)

## Cross-Asset Compatibility Check

- [x] Hooks section omitted — `enable_agent_extension_n8n_hooks = false`; hook wiring deferred until flag is enabled
- [x] `asset.yaml` `requires` entry with `kind: agent` references `sap.agtrefapp:agent:purchase-requisition-agent:v1` v1.0.14
- [x] `solution.yaml` contains refs to both:
  - `./assets/workflows/pr-task-center-notification-workflow/asset.yaml`
  - `./assets/pr-task-center-extension/asset.yaml`
- [x] No credentials or instance-specific values are hardcoded in any file
