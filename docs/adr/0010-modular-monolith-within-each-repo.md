# 0010. Modular Monolith Within Each Repo (Not Microservices Per Capability)

## Status

Accepted

## Context

Within the execution plane alone there are at least seven distinguishable capabilities: the LangGraph runtime, Temporal workflow definitions, the MCP/tool gateway, the RAG pipeline, memory services, the model gateway, and runtime policy enforcement (plus OTel instrumentation threaded through all of them). Within the control plane there are six: agent registry, YAML validation, model/tool/knowledge registries, policy authoring, deployment manager, evaluation manager. Each of these could, in principle, be its own microservice with its own deployable and its own repo. We needed to decide the granularity of decomposition beyond the two-plane split already decided in [ADR-0002](0002-control-execution-plane-separation.md).

## Decision

Within each plane, these capabilities are **internal modules of one deployable Python service**, not separate services or separate repos — a modular monolith per plane. The decomposition rule used throughout this project: a component is its own separately-deployed service only when it has a genuinely independent failure domain or scaling profile that justifies the operational cost of splitting it out; otherwise it's an internal module with a clean interface, living alongside the modules it most often changes together with.

## Alternatives Considered

- **One microservice per capability** (e.g. a separate deployable for the MCP gateway, a separate one for RAG, a separate one for memory, etc.). Rejected: these modules change together far more often than independently — a new tool type touches the MCP gateway and typically the LangGraph node that invokes tools; a new memory kind touches memory services and the graph's state schema; `human_in_loop` support touched Temporal workflows, policy enforcement, and graph signal-handling in one coherent change. Splitting these into separate services would turn most feature work into a multi-repo, multi-deploy coordination exercise for no isolation benefit, since none of these modules has an independent scaling profile worth paying that operational tax for.
- **A single repo/service for the entire platform** (folding the control plane into the execution plane too, undoing [ADR-0002](0002-control-execution-plane-separation.md)). Rejected — the two planes genuinely do have independent scaling and failure profiles, unlike the sub-capabilities within each plane; that's precisely the distinction this ADR is drawing (split where isolation is earned, not by default in either direction).

## Consequences

**Positive:** feature work that spans multiple related capabilities (the common case) is a single-repo, single-deploy change; no premature network boundaries between modules that always change together; each plane still gets independent deployability, scaling, and failure isolation from the other plane, which is where the real isolation need actually is.

**Negative / failure modes:** a bug in one module (e.g. a memory-service defect) shares a failure domain with the rest of its plane's process — a crash in the execution plane's process takes down LangGraph, Temporal-adjacent code, the MCP gateway, RAG, memory, and the model gateway together, not just the failing module. This is an accepted tradeoff since none of these modules is expected to fail independently often enough to justify isolating it, and Temporal's durability guarantees ([ADR-0004](0004-temporal-for-durable-execution.md)) already handle the "the whole execution-plane process died" case for `durable`/`human_in_loop` executions regardless of which internal module caused it.

**Consistency/state:** internal module boundaries are enforced by code organization and interface discipline (clean module boundaries, no reaching into another module's internals), not by network calls or separate transactions — this is weaker enforcement than the plane boundary (which is enforced by the repo split and, at the data layer, by Postgres role grants per [ADR-0006](0006-timescaledb-single-database.md)), and is an explicit tradeoff: strong enforcement is reserved for the boundary that actually needs it.

**Scaling:** each plane scales as a unit — adding execution-plane capacity scales all of its internal modules together, which is acceptable because their load is correlated (an agent invocation touches most of them together) rather than independent. If a specific module's load ever decorrelates sharply from the rest of its plane (e.g. RAG ingestion becoming a distinct heavy batch workload), that would be the concrete trigger to reconsider splitting it out — not speculative future need, an actual observed decorrelation, consistent with the "split only where isolation is earned" rule this ADR establishes.
