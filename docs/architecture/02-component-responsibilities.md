# 02. Component Responsibilities

This document enumerates every major component in AgentForge, states its single responsibility, and names which repository owns it. The rule used to decide "is this one component or two" throughout is: a component is the smallest unit that has one reason to change. If two pieces of behavior would always be modified together, they're one component; if they can evolve independently, they're separate — even if they currently live in the same repo (see [doc 06](06-repository-structure.md) and [ADR-0010](../adr/0010-modular-monolith-within-each-repo.md) for how that principle maps onto repo boundaries specifically).

## Component diagram

```mermaid
flowchart TB
    subgraph FE["agentforge-frontend"]
        UI[Developer Portal<br/>Next.js App Router]
    end

    subgraph CP["agentforge-control-plane (FastAPI)"]
        AR[Agent Registry]
        VAL[YAML Validator<br/>semantic layer]
        MR[Model Registry]
        TR[Tool Registry]
        KR[Knowledge Registry]
        PE_AUTHOR[Policy Engine<br/>authoring]
        DM[Deployment Manager]
        EM[Evaluation Manager]
        MIG[Alembic Migrations]
    end

    subgraph EP["agentforge-agent-execution-platform"]
        LG[LangGraph Runtime]
        TMP[Temporal Workflows]
        MCP[MCP / Tool Gateway]
        RAG[RAG Pipeline]
        MEM[Memory Services]
        MG[Model Gateway]
        PE_ENFORCE[Policy Engine<br/>runtime enforcement]
        OTEL[OTel Instrumentation]
    end

    subgraph SCHEMA["agentforge-agent-schema"]
        JSCH[agent.schema.json]
    end

    subgraph DB[("agentforge — TimescaleDB")]
        CPT[Control-plane tables]
        EPT[Execution-plane tables<br/>+ execution_steps hypertable]
    end

    UI -->|REST /api/v1| CP
    UI -->|SSE/WS + approvals| EP
    CP -->|DDL + CRUD| CPT
    EP -->|scoped role, no DDL| EPT
    EP -->|registry lookups at deploy/start| CP
    VAL --> JSCH
    LG --> MCP
    LG --> RAG
    LG --> MEM
    LG --> MG
    TMP -.wraps.-> LG
    PE_ENFORCE --> TMP
    CP -.depends on.-> JSCH
    FE -.depends on.-> JSCH
```

## Control plane components (`agentforge-control-plane`)

**Agent Registry** — owns the `agents` and `agent_versions` tables. Responsible for CRUD on agent metadata and immutable versioning: every save creates a new `agent_versions` row rather than mutating an existing one, so a deployed version can never change under a running deployment.

**YAML Validator (semantic layer)** — the second of the two validation layers described in [doc 07](07-agent-yaml-schema.md). Given YAML that has already passed JSON Schema structural validation, checks that every referenced `tool_id`, `model_id`, `knowledge_base_id`, and `policy_ref` actually exists and that the calling user/tenant is permitted to use it. This layer can only live in the control plane because it's the only component with registry access.

**Model Registry, Tool Registry, Knowledge Registry** — each owns one table (`models`, `tools`, `knowledge_bases` + `documents`) and the CRUD API for it. These are intentionally thin: they hold metadata (a tool's input/output JSON Schema, a model's provider and model ID, a knowledge base's source type and indexing status), not runtime logic. Runtime logic for actually calling a tool or model lives in the execution plane.

**Policy Engine (authoring half)** — owns the `policies` table and the CRUD API for declarative policy rules (`GET/POST /policies`, `GET/PATCH /policies/{id}`). Authoring is deliberately separated from enforcement: the control plane's job is to let a platform admin define *what* the rules are; the execution plane's job (see below) is to *apply* those rules to a live tool-call request at run time. See [ADR-0008](../adr/0008-mcp-tool-abstraction-and-policy-engine.md).

**Deployment Manager** — owns the `deployments` table. Promotes a specific `agent_version` to an environment, tracks status, and handles rollback (`POST /agents/{id}/deployments/{id}/rollback`). Deployment is a control-plane concern precisely because it's about *which version is the current definition*, not about any in-flight execution.

**Evaluation Manager** — owns `evaluation_suites` and orchestrates eval runs (`eval_runs`, `eval_results` are written by the execution plane as evaluations actually execute, but the suite definition and triggering — including `run_on_deploy` — are control-plane concerns).

**Alembic migration chain** — the sole owner of schema DDL for the entire `agentforge` database, including tables the execution plane writes rows to. See [doc 08](08-database-schema.md) and [ADR-0006](../adr/0006-timescaledb-single-database.md) for why this asymmetric ownership exists and how it's enforced.

## Execution plane components (`agentforge-agent-execution-platform`)

**LangGraph Runtime** — owns the reasoning graph: nodes (LLM calls, tool calls, routing/conditional edges), in-execution agent state, and multi-agent handoff within a single run (used by the Supervisor agent in Phase 9). This is the component that actually "thinks." See [doc 05](05-langgraph-vs-temporal.md) for the precise boundary between this and Temporal.

**Temporal Workflows** — owns durability across time and failure: retries with backoff, timers, human-approval signal-waits, crash recovery via event-sourced replay, and coordination of long-running external side effects. Wraps a LangGraph invocation as a unit for `durable` and `human_in_loop` execution modes; absent entirely for `ephemeral` mode.

**MCP / Tool Gateway** — the single choke point through which every tool call passes, regardless of which agent or which tool. Resolves a `tool_id` to an MCP server connection, enforces the permission (`read`/`write`/etc.) declared in the agent's YAML, and is where the Policy Engine's runtime half intercepts a request before it reaches the external system. See [ADR-0008](../adr/0008-mcp-tool-abstraction-and-policy-engine.md).

**RAG Pipeline** — ingestion (chunking, embedding, indexing documents into OpenSearch) and retrieval (hybrid search at query time) for knowledge bases referenced by an agent's `knowledge` section. See [ADR-0007](../adr/0007-opensearch-hybrid-retrieval.md).

**Memory Services** — working memory (single-execution scratchpad), episodic memory (past-execution recall), and semantic memory (durable facts/preferences), each toggled independently per-agent via the `memory` section of the YAML.

**Model Gateway** — a provider-abstraction layer in front of LLM calls, so an agent's YAML names a `provider` (`bedrock`/`mock`/`openai`) and the execution plane routes to the right client without the LangGraph graph code needing to know provider-specific request/response shapes. See [ADR-0005](../adr/0005-bedrock-primary-with-provider-abstraction.md).

**Policy Engine (runtime enforcement half)** — evaluates a tool-call or state-transition request against the policies referenced in the agent's `permissions.policy_refs`, returning `ALLOW` / `DENY` / `HUMAN_APPROVAL`. Every decision is written to `policy_decisions`. A `HUMAN_APPROVAL` result is what causes a Temporal workflow to create an `approvals` row and pause on a signal-wait.

**OTel Instrumentation** — emits spans/metrics across both LangGraph node execution and Temporal activity execution, correlated by `otel_span_id` stored on `execution_steps`, and exported to Prometheus/Grafana. See [doc 05's](05-langgraph-vs-temporal.md) note on why this is threaded through both subsystems rather than owned by just one.

## Shared contract (`agentforge-agent-schema`)

**`agent.schema.json`** — the versioned JSON Schema that defines what a legal agent YAML document looks like. Not a "component" in the runtime sense — it ships no server, no process — but it is depended on by three of the other repos (frontend, for Monaco autocomplete/inline diagnostics; control plane, for structural validation; execution platform, indirectly, since it only ever executes YAML that has already passed this schema) via pinned git dependencies, which is why it is its own repository rather than living inside any one of them. See [doc 07](07-agent-yaml-schema.md).

## Frontend components (`agentforge-frontend`)

The frontend is a thin, purely presentational client over the two planes' APIs — it holds no independent business logic (no client-side re-implementation of validation rules, policy evaluation, etc.). Its component inventory is covered in full in [doc 10](10-frontend-structure.md).

## Cross-cutting: what "component" does not mean here

None of the components above are separate deployables except at the two top-level boundaries (the control-plane FastAPI process and the execution-platform process, plus the frontend and the database). Within `agentforge-control-plane` and within `agentforge-agent-execution-platform`, these are internal modules — a deliberate choice (a modular monolith per plane, not a constellation of microservices per capability) explained in [ADR-0010](../adr/0010-modular-monolith-within-each-repo.md).
