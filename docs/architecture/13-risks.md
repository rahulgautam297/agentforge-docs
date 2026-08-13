# 13. Risks and Deferred Hardening

This document collects the risks, deliberately deferred hardening items, and open edges that come up across the rest of this documentation set, in one place, so a reader (or the project's own future self) doesn't have to hunt through every doc to find "what did we decide not to do yet, and why."

## Deferred hardening (deliberate, not oversights)

### Postgres Row-Level Security
Multi-tenancy is currently enforced entirely at the API layer — every query in both planes is scoped by the tenant resolved from the caller's bearer token (see [doc 08](08-database-schema.md#multi-tenancy) and [doc 03](03-control-plane.md#multi-tenancy)). This is sufficient for a project with no untrusted multi-tenant traffic, but it means a bug in API-layer scoping code is not caught by a second, database-enforced layer. RLS is the natural next step and is explicitly deferred to around Phase 11, when production infrastructure work is already reassessing the platform's trust boundaries.

### Read replicas / connection pooling
A single TimescaleDB primary serves both planes throughout the documented roadmap. If either plane's read load outgrows a single primary, PgBouncer-style pooling or read replicas are the next lever — not built now because no phase in the current roadmap produces load anywhere near that threshold, and building it prematurely would add operational complexity with no present benefit. Revisit if Phase 11's infrastructure work or any later scale-testing shows otherwise.

### `execution_steps` retention/compression policy
The hypertable partitioning described in [doc 08](08-database-schema.md) is in place from Phase 1, but no concrete compression or retention policy (e.g., "compress chunks older than 30 days, drop after a year") is defined yet — there isn't yet enough real usage data to size a sensible window against. This should be revisited once the platform has run long enough to have an actual `execution_steps` growth curve to look at.

### Sibling-directory assumption in local Compose
`agentforge-infra`'s relative build contexts (`../agentforge-frontend`, etc.) hard-assume sibling directory placement (see [workspace-setup.md](../../workspace-setup.md)), with no environment-variable override. This is a real rough edge for anyone who clones the repos into a different layout — low cost to fix (parameterize build context paths) but not yet done since Phase 0–1 has exactly one, known-good local layout in practice.

### No complex policy condition DSL
The Policy Engine starts with a small declarative rule set (name → rule as `jsonb`, evaluated to `ALLOW`/`DENY`/`HUMAN_APPROVAL`), not a general-purpose condition language. This is intentional simplicity for the phases in scope — see [ADR-0008](../adr/0008-mcp-tool-abstraction-and-policy-engine.md) — but it's a real limitation if policy needs grow more conditional (e.g., "allow if X and (Y or not Z)") than the initial rule shape comfortably expresses; revisit if Phase 9's Fraud Investigation Agent's real-world policy needs turn out to demand it.

## Risks tied to specific architectural choices

### Temporal server as a durability single point of failure
As discussed in [doc 05](05-langgraph-vs-temporal.md#failure-modes-unavailability-and-recovery-by-mode), `durable` and `human_in_loop` executions cannot make progress if Temporal's own server/cluster is unavailable — they are durably paused, not lost, but "durably paused" is only good enough if Temporal itself recovers in a reasonable timeframe. Locally this is a low-stakes risk (a single dev-server container); in production this becomes a real operational dependency worth monitoring and alerting on independently from the two AgentForge planes.

### Control-plane-down blocks authoring, not execution — but only if the caveat holds
[Doc 03](03-control-plane.md#failure-modes-and-unavailability) and [doc 04](04-execution-plane.md#failure-modes-and-unavailability) both state that in-flight executions are unaffected by control-plane downtime, with one caveat: an execution that's at the very first moment of starting still needs one synchronous registry-lookup call to the control plane. If the control plane has an extended outage exactly when a burst of new executions is expected to start, that burst is blocked, even though everything already running keeps running. This is an accepted tradeoff, not a bug, but worth stating plainly rather than letting "execution plane is independent" be read as an absolute claim.

### Two-database design was considered and rejected — worth remembering why
[ADR-0006](../adr/0006-timescaledb-single-database.md) documents that an earlier draft split control-plane and execution-plane data into two physically separate Postgres databases, and that this was explicitly reversed in favor of one database with role-based access control enforcing the plane boundary instead. This isn't a risk in the sense of something currently wrong, but it's worth keeping visible in case future scaling pressure ever makes a physical split look attractive again — the original reasoning against it (redundant operational cost, loss of real foreign keys, polyrepo already providing the plane-separation boundary) should be revisited on its merits at that time, not silently reversed by drift.

### Polyrepo cross-repo versioning latency
As noted in [doc 06](06-repository-structure.md#cross-repo-versioning), a change that spans the schema repo and a dependent repo requires sequencing (tag the schema first, then bump the pin) rather than an atomic monorepo commit. For a solo-maintained project this is a minor friction cost; it would become a more significant one if the project ever had multiple concurrent contributors making frequent cross-cutting schema changes. See [ADR-0001](../adr/0001-polyrepo-vs-monorepo.md) for the full tradeoff.

### `additionalProperties: false` strictness cuts both ways
The strict JSON Schema validation described in [doc 07](07-agent-yaml-schema.md) catches typos and drift, which is the point — but it also means every legitimate new field requires an explicit schema change and `schema_version` bump before any agent can use it, even experimentally. This is the correct tradeoff for a governed platform (see the schema doc's reasoning), but it does mean the schema repo is on the critical path for any new agent capability, which is worth remembering when planning future phases that add new YAML sections.

## Non-risks worth naming explicitly

To be clear about what is *not* considered an open risk in this design: the LangGraph/Temporal boundary ([doc 05](05-langgraph-vs-temporal.md)) is considered settled, not provisional — it's based on the two frameworks' actual, disjoint design purposes, not a placeholder pending more research. Similarly, the choice of a single TimescaleDB database ([ADR-0006](../adr/0006-timescaledb-single-database.md)) is a considered reversal of an earlier draft, not an unresolved question — it should not be re-litigated without new information beyond what's already weighed in that ADR.
