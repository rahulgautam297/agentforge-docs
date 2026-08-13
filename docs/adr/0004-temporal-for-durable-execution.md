# 0004. Temporal for Durable Execution

## Status

Accepted

## Context

Some agent executions are not safe to simply lose and retry from scratch on a crash: `durable`-mode executions may include non-idempotent side effects and can span wall-clock time beyond one request; `human_in_loop`-mode executions must pause — potentially for hours — waiting on a human decision, without holding process resources hostage while waiting, and must resume correctly regardless of how long the platform's own processes have been restarted or redeployed in the meantime. We needed a mechanism to provide retries with backoff, timers, crash-recoverable execution via replay, and long-duration external signal waits, layered around (not replacing) the LangGraph reasoning graph described in [ADR-0003](0003-langgraph-for-agent-orchestration.md).

## Decision

Use **Temporal** as the durable-execution layer. Temporal workflows wrap a LangGraph invocation as a unit for `durable` and `human_in_loop` execution modes; `ephemeral`-mode executions have no Temporal involvement at all. Temporal owns retries/backoff (per the agent's `execution.retry_policy`), timers (e.g. approval `timeout_seconds`), human-approval signal-waits, and crash recovery via event-sourced replay. The explicit rule, detailed in [doc 05](../architecture/05-langgraph-vs-temporal.md): Temporal wraps the graph invocation (or specific durable sub-steps, like a rollback-and-verify side effect), never every individual LLM call or tool call as its own activity.

## Alternatives Considered

- **Custom retry/checkpoint logic on top of a job queue** (e.g. Celery/RQ plus hand-rolled state persistence for resumability). Rejected: reimplements a large, subtle piece of distributed-systems machinery (exactly-once side-effect execution, deterministic replay, long-duration signal waits) that Temporal already provides correctly and battle-tested; a hand-rolled version is a significant, ongoing correctness risk for a solo project.
- **Step Functions or a similar cloud-native workflow service.** Rejected for Phase 0–9 scope specifically because it would tie local development (see [doc 11](../architecture/11-local-development.md)) to a cloud dependency this project explicitly avoids needing until Phase 10; Temporal's official dev-server container gives full production-equivalent semantics locally with zero cloud credentials.
- **Wrapping every LLM/tool call in its own Temporal activity.** Considered and explicitly rejected as an anti-pattern — see [doc 05](../architecture/05-langgraph-vs-temporal.md#the-anti-pattern-this-rule-exists-to-prevent) for the full reasoning: added per-call latency/overhead for no durability benefit on cheap-to-redo work, and fragmentation of a coherent reasoning graph into an unnecessarily granular activity soup.

## Consequences

**Positive:** `durable`/`human_in_loop` executions survive execution-plane process crashes and redeploys without losing progress or double-executing already-completed side effects; approval waits cost nothing while pending, however long they last; retry/backoff behavior is declarative (via the agent's YAML) rather than hand-coded per agent.

**Negative / failure modes:** Temporal's own server becomes a real durability dependency — if it's unavailable, no `durable`/`human_in_loop` workflow can make progress (though nothing is lost; they're durably paused) until it recovers. This is a genuine operational risk worth monitoring independently, tracked in [doc 13](../architecture/13-risks.md).

**State persisted:** Temporal's server persists a full event-sourced history of each workflow, separate from the `agentforge` database; AgentForge's own `executions`/`execution_steps`/`approvals` rows are a parallel, application-level record written by the execution plane as it progresses, eventually consistent with Temporal's internal replay state (replay does not re-execute already-completed side effects or duplicate already-written rows).

**Consistency model:** exactly-once side-effect execution per Temporal's own guarantees for wrapped activities; AgentForge's database writes happen once per step regardless of internal replay count.

**Scaling:** Temporal workers scale horizontally behind Temporal's task queue; the platform's genuine scaling pressure point remains `execution_steps` write volume, addressed by the hypertable design in [ADR-0006](0006-timescaledb-single-database.md), not by Temporal itself.
