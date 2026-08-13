# 10. Frontend Structure

## Purpose and stance

`agentforge-frontend` is a Next.js (App Router) + TypeScript + Tailwind + Monaco + React Query developer portal — the human-facing surface over both planes' APIs. It deliberately holds **no independent business logic**: it does not re-implement JSON Schema structural validation, does not re-implement semantic validation, does not re-implement policy evaluation. Every "is this allowed" decision is answered by calling the control plane or execution plane and rendering their response; the frontend's job is presentation, editing ergonomics, and live-updating views over server state, not decision-making. This keeps validation/policy logic in exactly one place per concern (matching the platform's core rule of never hardcoding agent-specific or decision-specific logic outside its owning component) and means the frontend can be rebuilt or replaced without touching platform behavior.

## Pages

| Route | Purpose |
|---|---|
| `/dashboard` | Landing overview — recent executions, pending approvals, agent/deployment health at a glance. |
| `/agents` | List of agents (tenant-scoped), with status badges. |
| `/agents/new` | Agent authoring: Monaco YAML editor on the left, live config preview on the right, Validate/Save/Deploy actions, inline validation errors surfaced at the exact field (e.g. "Tool kubernetes.write is not allowed by current policy" attached to the offending `tools[]` entry). |
| `/agents/:id` | Agent detail — metadata, version history, deployment history. |
| `/agents/:id/executions` | Execution/trace viewer — list of past executions, each expandable into its full `execution_steps` waterfall/tree. |
| `/knowledge` | Knowledge base registry — list, create, manage documents. |
| `/tools` | Tool registry — list, create. |
| `/models` | Model registry — list, create. |
| `/evaluations` | Evaluation suite management, trigger runs, view results. |
| `/approvals` | Pending human-in-the-loop approval requests, rendered as cards. |
| `/settings` | Tenant/user settings. |

## `/agents/new` in detail

This is the highest-stakes page in the frontend, since it's where a developer's intent becomes a validated, deployable artifact. The two-pane layout (Monaco editor left, live rendered preview right) exists so a developer can see the *effect* of their YAML — which tools, which policies, which approval checkpoints — without mentally parsing raw YAML structure. Monaco is wired directly to `agent.schema.json` (from `agentforge-agent-schema`, pinned as a dependency — see [doc 06](06-repository-structure.md)) for autocomplete and inline diagnostics purely client-side, catching the same structural issues Layer 1 validation would catch, before the developer even hits Validate. The Validate action then calls the control plane's `POST /agents/validate` (stateless, since the agent may not be saved yet) to run both validation layers, including the semantic layer that only the control plane can perform — e.g. confirming `kubernetes.write` isn't actually permitted, which is exactly the example error shown in the page's own description because it demonstrates why client-side schema checking alone is insufficient: the schema layer only knows `write` is a legal permission enum value, not whether *this tenant's current policy* allows it for *this tool*.

## Shared components

- **`YamlEditor`** — Monaco instance configured with the agent schema for autocomplete and inline diagnostics. Used on `/agents/new` and version-diff views.
- **`YamlPreviewPane`** — renders parsed YAML as a structured, human-readable summary (model, tools + permissions, approval checkpoints, etc.) rather than raw text, so a reviewer can sanity-check an agent's capabilities at a glance.
- **`TraceViewer`** — renders the nested `execution_steps` tree fetched from `GET /executions/{id}/trace` as a waterfall/tree, each step expandable to show its `tool_calls`/`model_calls` detail (request/response payloads, latency, token counts). Consumes the SSE stream for in-progress executions so a running trace fills in live.
- **`ApprovalCard`** — renders one pending approval: agent name, the requested action, the stated reason, and Approve/Reject controls that call `POST /approvals/{id}/decision`.
- **`AgentStatusBadge`** / **`ExecutionStatusBadge`** — small status indicators reused across list and detail views, keeping status-to-color/label mapping in one place rather than duplicated per page.

## Data fetching

React Query owns all server-state caching, invalidation, and refetch-on-focus behavior — list pages (`/agents`, `/tools`, `/models`, `/knowledge`, `/evaluations`, `/approvals`) are ordinary query-and-cache reads; mutations (validate/save/deploy/execute/approve) invalidate the relevant query keys so, e.g., approving a request immediately removes it from `/approvals`'s pending list without a manual refetch. Live execution progress (the TraceViewer while an execution is running, and the dashboard's "recent executions" panel) layers an SSE subscription on top of the initial React Query-fetched snapshot rather than polling — appropriate given the execution plane already emits a step-by-step event stream for exactly this purpose (see [doc 09](09-api-contract.md#streaming-and-the-approval-loop)).

## Which plane the frontend talks to, and why it doesn't matter to the frontend

The frontend calls both the control plane (agents, versions, validation, deployments, tools, models, knowledge, evaluations, policies) and the execution plane (executions, trace, approvals) under the same `/api/v1` convention. The frontend's data layer treats this as one logical API — it does not encode plane awareness into its own code (no `if (controlPlaneDown)` branches) — but the *user-visible consequence* of plane availability differs by page, and this is worth being explicit about since it's a direct, visible manifestation of the control/execution plane split described in [doc 01](01-overview.md): if the control plane is down, `/agents`, `/agents/new`, `/tools`, `/models`, `/knowledge`, `/evaluations`, `/settings` become read/write-unavailable, while `/agents/:id/executions` and `/approvals` keep working because they're served by the execution plane. The reverse holds if the execution plane is down: authoring pages keep working, but no new executions can start and live trace streaming stops.

## Stack rationale

**Next.js App Router** for its file-system routing matching the page list above almost 1:1, and for React Server Components reducing client bundle size on data-heavy pages like the trace viewer. **TypeScript** throughout, with types generated from the control plane's OpenAPI schema (FastAPI's automatic OpenAPI generation) so request/response shapes stay in sync with the backend without hand-maintained types drifting out of date. **Tailwind** for consistent, low-overhead styling across a fairly large page inventory. **Monaco** specifically (rather than a lighter-weight code editor) because it's the same editor VS Code uses and has mature JSON Schema-driven YAML language support out of the box — exactly the autocomplete/diagnostics behavior `/agents/new` depends on. **React Query** for server-state management, chosen over a heavier global-state library (Redux, etc.) because almost everything the frontend renders *is* server state (agents, versions, executions, approvals) with comparatively little genuinely client-only UI state, which is exactly React Query's sweet spot.

## Failure modes

The frontend has no server-side state of its own to lose — a frontend crash or redeploy loses nothing durable, since everything it shows is re-fetched from the two planes. Its own unavailability means developers can't use the portal, but has zero effect on already-running executions or already-deployed agents, consistent with the platform-wide principle that presentation-layer downtime never threatens execution-plane durability guarantees.
