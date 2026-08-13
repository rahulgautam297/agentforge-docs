# 0003. LangGraph for Agent Orchestration

## Status

Accepted

## Context

AgentForge needs a way to express an agent's actual reasoning: LLM calls, tool calls, conditional routing between them based on model output, in-execution state accumulation, and — for the Supervisor agent (Phase 9) — delegation to and synthesis from sub-agents within one run. This needs to be generic across every agent the schema can describe, not hand-coded per agent (see [doc 01](../architecture/01-overview.md)'s core principle). We needed an orchestration library for this reasoning layer, distinct from (and composed with, not replaced by) whatever handles durability across time and failure — that separate concern is addressed in [ADR-0004](0004-temporal-for-durable-execution.md).

## Decision

Use **LangGraph** as the reasoning-graph engine: nodes for LLM calls, tool calls, and routing; conditional edges for control flow; typed state passed between nodes; subgraphs/multi-agent patterns for the Supervisor's delegation. LangGraph owns exactly this layer and nothing else — it has no role in durability, retries beyond its own graph-level concerns, or human-approval waits spanning wall-clock time; that's Temporal's job, detailed in [doc 05](../architecture/05-langgraph-vs-temporal.md).

## Alternatives Considered

- **Hand-rolled state machine / custom orchestration code.** Rejected: reinvents a well-trodden problem (typed state passing, conditional routing, streaming, tool-calling conventions) that LangGraph already solves, for a project whose value is in demonstrating platform-engineering judgment on the *harder* problems (governance, durability, observability), not in reinventing graph orchestration.
- **Encoding the graph directly as Temporal workflow code.** Rejected: Temporal has no purpose-built abstractions for LLM reasoning graphs (typed state, conditional routing on model output, prebuilt agent patterns) — implementing this directly in Temporal workflow code would mean losing all of that for no benefit, since Temporal doesn't compete with LangGraph on this axis at all. See [doc 05](../architecture/05-langgraph-vs-temporal.md) for the full boundary argument.
- **A different agent framework** (e.g. a simpler linear-chain library with no graph/conditional-routing support). Rejected: several showcase agents (notably the Supervisor and the Incident Investigator's branching investigation flow) need genuine conditional routing and multi-agent handoff within one execution, which a linear-chain abstraction doesn't naturally express.

## Consequences

**Positive:** a mature, purpose-built abstraction for exactly the reasoning-graph shape every AgentForge agent needs; supports the full range from a single-node echo agent (Phase 1) to the Supervisor's multi-agent delegation (Phase 9) with the same underlying engine.

**Negative / failure modes:** LangGraph's own checkpointing is not a substitute for Temporal's durability guarantees (no exactly-once side-effect execution, no arbitrary-duration external signal waits, no replay-based recovery across a full process crash) — an `ephemeral`-mode execution that crashes mid-graph is simply lost, by design; anything needing to survive that must be wrapped in Temporal ([ADR-0004](0004-temporal-for-durable-execution.md)).

**State persisted:** in-execution graph state lives in process memory during a run and is not itself the durability mechanism — persisted trace data (what actually happened) is written to `execution_steps` as the graph progresses, independent of LangGraph's own state representation.

**Scaling:** LangGraph invocations are stateless-per-process beyond what's explicitly checkpointed, so execution-plane workers scale horizontally without LangGraph itself imposing a scaling bottleneck; the real scaling pressure point is the trace-write volume this produces, addressed by the hypertable design in [ADR-0006](0006-timescaledb-single-database.md).
