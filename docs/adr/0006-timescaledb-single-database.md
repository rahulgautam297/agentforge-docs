# 0006. Single TimescaleDB Database (Not Split by Plane)

## Status

Accepted — **this ADR supersedes an earlier draft plan** that proposed two physically separate Postgres databases, one per plane. That earlier draft is documented below, in Alternatives Considered, precisely because it was seriously considered and explicitly reversed during planning, not because it's a strawman.

## Context

AgentForge has two planes (see [ADR-0002](0002-control-execution-plane-separation.md)) with an asymmetric data-access requirement: the control plane needs full schema-migration (DDL) authority; the execution plane needs read/write access to a specific set of tables and must never be able to alter schema. An earlier draft, produced during initial planning, proposed enforcing this asymmetry by giving each plane its own physically separate Postgres database. That draft was presented to the project owner and **explicitly rejected** in favor of one shared database, on the reasoning that the polyrepo split (see [ADR-0001](0001-polyrepo-vs-monorepo.md)) already provides the plane-separation boundary — separate codebases, separate CI, separate deploys — making a second physical database redundant operational cost, and that real foreign keys in one database beat the value-only cross-database references the two-database design would have required for relationships like `execution → deployment → agent_version`.

Separately from the two-vs-one-database question, AgentForge needed a decision about how to store `execution_steps`, the trace/span table — extremely high-volume, strictly time-ordered, and the backing data for the custom TraceViewer.

## Decision

**One database**, named `agentforge`, running **TimescaleDB** (PostgreSQL plus the Timescale extension, fully Postgres-compatible). Every table lives in this one database. The plane boundary that the earlier two-database draft was trying to enforce is instead enforced by **Postgres role-based access control**: `agentforge-control-plane` connects with a role holding full DDL rights and is the sole owner of the Alembic migration chain; `agentforge-agent-execution-platform` connects with a separate, scoped role holding `SELECT`/`INSERT`/`UPDATE` grants only on the execution-owned tables (`executions`, `execution_steps`, `tool_calls`, `model_calls`, `policy_decisions`, `approvals`, `eval_runs`, `eval_results`) and zero DDL rights anywhere.

Within this one database, `execution_steps` specifically is modeled as a **Timescale hypertable**, auto-partitioned by `started_at` (e.g. 1-day chunks). Every other table is a normal relational table.

## Alternatives Considered

- **Two physically separate Postgres databases, split by plane (the rejected earlier draft).** Rejected because: (1) the polyrepo boundary already provides the plane-separation guarantee this was meant to add, making it redundant; (2) it forces every cross-plane relationship (`executions.deployment_id`, `executions.agent_version_id`) to become a value-only reference with no real foreign key, no referential integrity, and no ability to write a single join query across the boundary — exactly the relationships a trace viewer or audit investigation needs most; (3) it doubles operational surface (two connection pools, two backup/restore stories, two places migrations could drift) for no additional isolation benefit beyond what role-based access control already provides within one database.
- **ClickHouse for `execution_steps` specifically.** A genuinely strong OLAP engine for this workload, but rejected because it would introduce an entirely new query language and operational surface (separate database technology, separate tooling/monitoring/backup story) for one table's worth of data, when Timescale's hypertable partitioning gets most of the same benefit while staying fully SQL- and Postgres-tooling-compatible with the rest of the schema.
- **Grafana Tempo for trace storage.** Idiomatic for OpenTelemetry-native trace storage, but rejected because it stores raw traces separately from the structured `execution_steps` SQL table the custom TraceViewer API actually queries (joined against `tool_calls`/`model_calls`/`approvals`) — adopting it would mean maintaining two parallel read paths for "what did this execution do" instead of one. OTel spans are still emitted and correlated via `otel_span_id`; Tempo is not adopted as the trace-of-record store.

## Consequences

**Positive:** real foreign keys and joinable relationships across the entire schema, including across the plane boundary; one operational surface (one connection pool story, one backup/restore story, one migration chain) instead of two; the plane boundary is still genuinely enforced — by Postgres itself, via role grants, not merely by convention or code review discipline.

**Negative / failure modes:** the plane boundary's enforcement now depends on role-grant configuration being correct — a misconfigured grant could theoretically let the execution-plane role touch control-plane tables or (worse) run DDL; this is mitigated by the grants being applied via the same Alembic migration chain that manages all other schema, keeping them under the same review discipline as everything else, but it's a real dependency on configuration correctness that the two-database draft would have made structurally impossible instead of merely policy-enforced.

**State persisted / consistency model:** ordinary Postgres ACID transactions throughout; no eventual-consistency window between related writes; cross-plane relationships are enforced by real foreign-key constraints regardless of which role performed the insert.

**Scaling:** the hypertable partitioning on `execution_steps` is the primary scaling lever for the system's highest-volume table — chunk pruning keeps time-range and per-execution queries cheap as data grows, and old chunks are natural candidates for Timescale's compression/retention policies (not yet configured — see [doc 13](../architecture/13-risks.md)). A single TimescaleDB instance is expected to be sufficient through the full 11-phase roadmap; read replicas or connection pooling are deferred, tracked in [doc 13](../architecture/13-risks.md).

**What happens when unavailable:** since both planes share one database, a full outage of `agentforge` stops both planes' persistence simultaneously — there is no scenario where one plane's database is down and the other's is up, which is a simpler (if less isolated) failure mode than the two-database draft would have produced, and consistent with this decision's premise that the meaningful isolation boundary is the plane split itself ([ADR-0002](0002-control-execution-plane-separation.md)), not the database topology.
