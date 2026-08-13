# 0008. MCP Tool Abstraction and Policy Engine

## Status

Accepted

## Context

Agents need to call external systems (GitHub, Kubernetes, Prometheus, internal search, and more over time) without the platform hardcoding per-tool integration logic into agent-specific code — consistent with the platform's core principle (see [doc 01](../architecture/01-overview.md)) of never special-casing an agent. At the same time, some tool calls are dangerous enough (a production write, a rollback) that they must be gated by policy — sometimes outright denied, sometimes requiring human sign-off — and every such decision must be auditable after the fact. We needed both a uniform tool-calling abstraction and a uniform policy-decision mechanism that applies to any tool, any agent, without per-agent logic.

## Decision

Adopt **MCP (Model Context Protocol)** as the tool abstraction: every tool a mock or real integration exposes speaks MCP, and the execution plane's **MCP/Tool Gateway** is the single choke point every tool call passes through, regardless of agent or tool. Layer a **Policy Engine** in front of that gateway: every tool-call request is evaluated against the policies referenced in the calling agent's `permissions.policy_refs`, returning one of three outcomes — `ALLOW`, `DENY`, `HUMAN_APPROVAL` — with every decision written to `policy_decisions` for audit. Policy *authoring* (CRUD on policy rules) lives in the control plane; policy *enforcement* (evaluating a live request) lives in the execution plane, matching the plane split described in [ADR-0002](0002-control-execution-plane-separation.md). The initial policy rule set is small and declarative (a named rule as `jsonb`), not a general-purpose condition DSL.

Example decisions: `READ GitHub` → `ALLOW`; `READ production metrics` → `ALLOW`; `DELETE production resource` → `DENY`; `ROLLBACK production deployment` → `HUMAN_APPROVAL` (triggering the Temporal signal-wait described in [ADR-0004](0004-temporal-for-durable-execution.md) and [ADR-0009](0009-human-approval-mechanism.md)).

## Alternatives Considered

- **Per-tool, per-agent integration code** (each agent's graph directly calling a tool-specific SDK/client). Rejected outright: this is exactly the hardcoded-agent-logic anti-pattern the whole platform exists to avoid, and it gives policy enforcement no single choke point to sit in front of.
- **A general-purpose policy condition DSL from day one** (e.g. a full rule language supporting arbitrary boolean composition). Rejected for the current phases: adds real implementation and testing surface for conditions the current four showcase agents don't yet need; a small declarative rule set is sufficient through Phase 9, with the DSL question explicitly revisited if the Fraud Investigation Agent's real-world needs demand it (tracked in [doc 13](../architecture/13-risks.md)).
- **Policy enforcement in the control plane instead of the execution plane** (e.g. pre-computing an allow-list at deploy time). Rejected: some policy decisions genuinely depend on runtime context (what specific action is being requested, right now) that doesn't exist until execution time — pre-computing at deploy time would mean either over-approving or requiring the control plane to somehow anticipate every possible runtime request shape.

## Consequences

**Positive:** any new tool is a new MCP server implementation, not a new code path through the platform; policy enforcement is uniform and impossible to bypass by construction, since the gateway is the only path to any tool; every decision is auditable via `policy_decisions`.

**Negative / failure modes:** if the Policy Engine (or its dependency, the gateway) is unavailable, tool calls cannot proceed — this fails safe (no tool call reaches an external system without a decision), surfacing as an execution error or triggering `retry_policy` in `durable` mode, never as a silent bypass.

**State persisted:** every `policy_decisions` row records the policy applied, the execution, the decision, and a reason — an append-style audit trail, never mutated after the fact.

**Consistency model:** each policy decision is evaluated synchronously against the current state of the `policies` table at request time — a policy changed mid-execution by an admin takes effect on the *next* tool-call evaluation within that execution, not retroactively on already-decided calls.

**Scaling:** the gateway and policy evaluation are per-tool-call overhead, small and synchronous; the current declarative rule set is cheap to evaluate (no complex condition graph to traverse), keeping this a low-latency addition to the tool-call path even at the current simplicity level.
