# 0002. Control Plane / Execution Plane Separation

## Status

Accepted

## Context

AgentForge needs to both (a) let developers author, validate, deploy, and manage agent definitions, and (b) actually run those agents, observe them, and durably continue their execution across failures and human-approval waits. These two workloads have almost nothing in common operationally: (a) is low-volume, latency-insensitive, stateless request/response CRUD; (b) can be long-running (up to hours, for `human_in_loop` approval waits), holds meaningful in-flight state, and has fundamentally different failure semantics — a crash mid-execution must be recoverable, not just retriable by the client. We needed to decide whether one service handles both, or whether they're split.

An earlier draft additionally proposed backing this split with two *physically separate* Postgres databases (one per plane). That sub-decision is addressed in full, separately, in [ADR-0006](0006-timescaledb-single-database.md) — this ADR is about the plane split itself; ADR-0006 is about how the data layer represents it.

## Decision

Split into two planes, each its own deployable, each its own repository (see [ADR-0001](0001-polyrepo-vs-monorepo.md)): a **control plane** (`agentforge-control-plane`) managing agent definitions, registries, deployments, and policy authoring; and an **execution plane** (`agentforge-agent-execution-platform`) running agent reasoning, tool calls, and durable/human-gated executions. The control plane never blocks or is required for an execution already in flight — the execution plane never calls back into the control-plane HTTP API mid-run, only at execution start for a cached registry lookup. The plane boundary at the data layer is enforced by Postgres role-based access control against one shared database (see [ADR-0006](0006-timescaledb-single-database.md)), not by physical database separation.

## Alternatives Considered

- **One combined service.** Rejected: couples two workloads with incompatible scaling and failure profiles into one deployable, so scaling for slow agent executions would also scale pods doing trivial CRUD, and a bug in a validation endpoint could, in a shared process, threaten availability of live executions.
- **Physical database-per-plane** (the rejected earlier draft, detailed fully in [ADR-0006](0006-timescaledb-single-database.md)). Rejected because the polyrepo split already provides the plane-separation boundary, making a second physical database redundant operational cost that also destroys real foreign-key relationships across the plane line.
- **More than two planes** (e.g., separate services per registry, per workflow type). Rejected as premature decomposition — see [ADR-0010](0010-modular-monolith-within-each-repo.md) for the reasoning against splitting further within a plane.

## Consequences

**Positive:** each plane scales, deploys, and fails independently; control-plane downtime never threatens in-flight or newly-starting-but-already-past-the-initial-lookup executions; the boundary is real and enforceable (repo boundary + role-based data access), not just a naming convention.

**Negative / failure modes:** an execution that is at the exact moment of starting (needing its one synchronous control-plane registry lookup) will fail to start if the control plane is unreachable at that instant — this is a real, if narrow, coupling point, documented in [doc 13](../architecture/13-risks.md). Cross-plane relationships (e.g., an `execution` referencing the `deployment` it ran) rely on the single shared database's real foreign keys rather than an API call, which is precisely what ADR-0006 argues is the better tradeoff versus physical separation.

**Consistency model:** each plane's own writes are transactionally consistent within its own database role's tables; cross-plane relationships are enforced by ordinary Postgres foreign keys, since both planes' tables live in one database (see [ADR-0006](0006-timescaledb-single-database.md)).

**Scaling:** the execution plane is the platform's primary scaling concern, since its load is proportional to agent invocation volume; the control plane's load is proportional to developer activity, an order of magnitude smaller and independently scalable.
