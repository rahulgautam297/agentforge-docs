# 0009. Human Approval Mechanism

## Status

Accepted

## Context

Some agent actions — most vividly, the Incident Investigator proposing a production rollback — must not execute until a specific human (identified by role, e.g. `sre-oncall`) explicitly approves them. This wait can be arbitrarily long in principle (an on-call engineer might not see the request for a while) and must not consume compute resources while pending, must survive execution-plane process restarts, must auto-resolve (deny) if nobody responds within a bounded time, and every decision must be attributable to a specific user and auditable. We needed a concrete mechanism connecting the Policy Engine's `HUMAN_APPROVAL` outcome (see [ADR-0008](0008-mcp-tool-abstraction-and-policy-engine.md)) to an actual pause-and-resume in a running execution.

## Decision

Model human approval as a **Temporal signal-wait**, per the `human_in_loop` execution mode described in [ADR-0004](0004-temporal-for-durable-execution.md) and [doc 05](../architecture/05-langgraph-vs-temporal.md). When the Policy Engine returns `HUMAN_APPROVAL` for a tool-call request, the execution plane writes an `approvals` row (`status=pending`, `timeout_at` computed from the checkpoint's `timeout_seconds`) and the Temporal workflow pauses on a signal-wait — consuming no worker compute while paused. The frontend's `/approvals` page renders pending requests as `ApprovalCard`s (agent name, action, reason, Approve/Reject). `POST /approvals/{id}/decision` both updates the `approvals` row and signals the paused workflow to resume; if `timeout_at` lapses first, the workflow's own timer fires and applies the checkpoint's declared `on_timeout` behavior (`deny` in the schema's example — see [doc 07](../architecture/07-agent-yaml-schema.md)).

## Alternatives Considered

- **Polling-based approval** (execution plane periodically checks an `approvals` table for a decision, rather than a Temporal signal). Rejected: requires the execution plane to keep a worker (or a scheduled poll job) actively checking for however long the approval is pending, which is exactly the resource cost Temporal's signal-wait avoids; also loses Temporal's built-in timer semantics for the timeout case, requiring a separate timeout-tracking mechanism to be hand-built.
- **Synchronous HTTP long-poll / blocking request held open until a human responds.** Rejected outright: cannot survive an execution-plane process restart, and holding an HTTP connection open for a potentially hours-long wait is operationally untenable (proxy timeouts, connection pool exhaustion under load).
- **Approval as a separate, disconnected workflow** (an approval request that, once decided, triggers a brand-new execution rather than resuming the paused one). Rejected: loses the paused execution's accumulated context (everything the agent had already reasoned/gathered before proposing the action) — a resumed workflow continuing from exactly where it paused is what lets the Incident Investigator's post-approval steps (execute rollback, wait for deployment, verify recovery) use the same execution's context rather than starting over.

## Consequences

**Positive:** zero compute cost while an approval is pending, however long; correct behavior across execution-plane restarts (Temporal's durability guarantees, per [ADR-0004](0004-temporal-for-durable-execution.md), apply directly); automatic, declarative timeout handling via the checkpoint's `on_timeout` field, no hand-built timeout tracker needed; every decision is attributed to a specific `approver_user_id` and auditable via the `approvals` row.

**Negative / failure modes:** this mechanism only exists for `human_in_loop`-mode executions — an `ephemeral` or plain `durable` agent cannot use an approval checkpoint even if its YAML declared one, since there's no Temporal workflow to pause; the schema and semantic validation should (and does, at the design level) treat `approval.checkpoints` as only meaningful when `execution.mode: human_in_loop`, per [doc 07](../architecture/07-agent-yaml-schema.md). If Temporal itself is unavailable, pending approvals cannot be signaled to resume even if a human decides — the decision is recorded in `approvals` but the workflow resume is blocked until Temporal recovers (a specific instance of the broader risk noted in [doc 13](../architecture/13-risks.md)).

**State persisted:** `approvals` rows (`requested_at`, `status`, `approver_user_id`, `decided_at`, `comment`, `timeout_at`) in `agentforge`; the actual pause state lives in Temporal's own workflow event history, not in the `approvals` row itself — the row is AgentForge's application-level record of the same fact, kept consistent by the execution plane's own write discipline.

**Consistency model:** the `approvals` row and the Temporal workflow's paused state can, in principle, briefly diverge if a crash occurs between writing the row and entering the signal-wait (or between receiving a decision and signaling the workflow) — mitigated by ordering the row write before entering signal-wait, and by the decision endpoint being safe to retry (idempotent from the workflow's perspective — a duplicate signal for an already-resumed workflow is a no-op).

**Scaling:** approval checkpoints impose no meaningful load — they're inherently low-frequency (a human-paced action), and the signal-wait mechanism scales to arbitrarily many concurrently-pending approvals without per-approval worker cost.
