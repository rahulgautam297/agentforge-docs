# AgentForge Architecture

This is the complete architecture documentation for AgentForge, written to be thorough enough for a senior engineer to build Phase 1 from cold, and to teach *why* each choice was made — not just what it is. Read in order; each document builds on the ones before it.

## Reading order

| # | Document | Summary |
|---|---|---|
| 01 | [Overview](01-overview.md) | What AgentForge is, the problem it solves, the two-plane split at a glance, system context diagram. |
| 02 | [Component Responsibilities](02-component-responsibilities.md) | Every component in the system, its single responsibility, and which repo owns it. |
| 03 | [Control Plane](03-control-plane.md) | Deep dive: agent registry, validation, deployment, evaluation, policy authoring — and why control-plane downtime never stops in-flight executions. |
| 04 | [Execution Plane](04-execution-plane.md) | Deep dive: LangGraph, Temporal, MCP gateway, RAG, memory, model gateway, policy enforcement, OTel — and the three execution modes. |
| 05 | [LangGraph vs. Temporal](05-langgraph-vs-temporal.md) | The sharpest boundary in the system: reasoning graph vs. durability envelope, and the anti-pattern of wrapping every LLM call in a Temporal activity. |
| 06 | [Repository Structure](06-repository-structure.md) | The seven-repo polyrepo topology, dependency direction, and why polyrepo over monorepo. |
| 07 | [Agent YAML Schema](07-agent-yaml-schema.md) | The full agent YAML contract, section by section, and the two-layer (structural + semantic) validation model. |
| 08 | [Database Schema](08-database-schema.md) | The single shared `agentforge` TimescaleDB database, full ER diagram, table ownership, and the role-based plane boundary enforced at the data layer. |
| 09 | [API Contract](09-api-contract.md) | Every `/api/v1` endpoint, grouped by resource, with idempotency and streaming semantics. |
| 10 | [Frontend Structure](10-frontend-structure.md) | Pages, shared components, data-fetching strategy, and why the frontend holds no independent business logic. |
| 11 | [Local Development](11-local-development.md) | What `docker compose up` brings up, and every production dependency's local mock substitute. |
| 12 | [Implementation Plan](12-implementation-plan.md) | The 11-phase roadmap from this documentation package through a real-AWS, Kubernetes-deployed flagship demo. |
| 13 | [Risks](13-risks.md) | Deliberately deferred hardening, risks tied to specific choices, and non-risks worth naming explicitly. |

## Diagram legend and conventions

Diagrams throughout this documentation set use [Mermaid](https://mermaid.js.org/) and follow consistent conventions:

- **`flowchart`** diagrams show structural/component relationships — boxes are components or data stores, arrows show call or data-flow direction, labeled with the nature of the interaction where it isn't obvious.
- **`sequenceDiagram`** diagrams show time-ordered interactions between components for one specific scenario (e.g., create → validate → deploy, or a durable execution with a human approval checkpoint) — read top to bottom as a timeline.
- **`erDiagram`** diagrams show database entity relationships — one database, one diagram (see [doc 08](08-database-schema.md)), since AgentForge deliberately uses a single shared database rather than one per plane.
- **`C4Context`** is used once, in [doc 01](01-overview.md), for the system-context view — the widest-angle diagram in the set, showing AgentForge as a whole against external actors and systems.
- Dashed arrows (`-.->`) indicate an indirect, cached, or non-hot-path relationship (e.g., the execution plane's registry lookups into the control plane, made once at execution start rather than per step).
- Solid arrows (`-->`) indicate a direct, synchronous or primary-path relationship.

## Which ADRs back which decisions

Every major decision documented here has a corresponding [Architecture Decision Record](../adr/README.md) with the full alternatives-considered reasoning. Cross-references:

| Architecture doc | Backing ADR(s) |
|---|---|
| 01 Overview (two-plane split) | [ADR-0002](../adr/0002-control-execution-plane-separation.md) |
| 03 Control Plane / 04 Execution Plane (plane split) | [ADR-0002](../adr/0002-control-execution-plane-separation.md) |
| 05 LangGraph vs. Temporal | [ADR-0003](../adr/0003-langgraph-for-agent-orchestration.md), [ADR-0004](../adr/0004-temporal-for-durable-execution.md) |
| 04 Execution Plane (model gateway) | [ADR-0005](../adr/0005-bedrock-primary-with-provider-abstraction.md) |
| 06 Repository Structure | [ADR-0001](../adr/0001-polyrepo-vs-monorepo.md), [ADR-0010](../adr/0010-modular-monolith-within-each-repo.md) |
| 08 Database Schema | [ADR-0006](../adr/0006-timescaledb-single-database.md) |
| 04 Execution Plane (RAG) | [ADR-0007](../adr/0007-opensearch-hybrid-retrieval.md) |
| 04 Execution Plane (policy engine) / 05 LangGraph vs. Temporal | [ADR-0008](../adr/0008-mcp-tool-abstraction-and-policy-engine.md) |
| 04 Execution Plane (human-in-loop mode) | [ADR-0009](../adr/0009-human-approval-mechanism.md) |
| 02 Component Responsibilities (module boundaries) | [ADR-0010](../adr/0010-modular-monolith-within-each-repo.md) |

See the [ADR index](../adr/README.md) for the full list with status and one-line decisions.
