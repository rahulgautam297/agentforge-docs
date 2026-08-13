# AgentForge

**A self-service platform for defining, deploying, executing, observing, evaluating, and governing AI agents.**

AgentForge lets developers define AI agents declaratively in YAML — model, tools, knowledge sources, memory, execution mode, permissions, human-approval checkpoints, and evaluation suites — then validate, deploy, execute, and observe those agents through a control plane and execution plane that never hardcode agent-specific logic into the platform itself. An agent is data (a versioned YAML document validated against a JSON Schema contract), not a bespoke code path. The platform's job is to run *any* agent that conforms to the schema safely, observably, and governably.

This is a portfolio project. It is **inspired by** architectural patterns found in internal AI agent platforms at large regulated enterprises (fintechs in particular, where "an LLM autonomously touching production" has to clear approval workflows, audit trails, and policy engines before anyone will trust it) — it does not contain, copy, or reproduce any proprietary code, configuration, or design documents from any employer or company. Every decision documented here was made independently for this project, for the purpose of demonstrating platform-engineering judgment on a system whose problem shape (governed autonomy, human-in-the-loop control, durable execution, multi-tenant observability) is representative of that class of internal platform.

## Documentation

- **[Architecture](docs/architecture/README.md)** — the full system design: component responsibilities, control/execution plane split, LangGraph vs. Temporal boundary, agent YAML schema, database schema, API contract, frontend structure, local development, phased implementation plan, and risks.
- **[Architecture Decision Records](docs/adr/README.md)** — the individual decisions (and rejected alternatives) that the architecture is built from, in MADR-style records.
- **[Workspace Setup](workspace-setup.md)** — how the seven AgentForge repositories relate to each other and how to lay them out locally.

## Status

**Phase 0 — architecture and design documentation.** No application code exists yet. This repository, together with `agentforge-agent-schema`, is the complete Phase-0 deliverable: a documentation package thorough enough for a senior engineer to build Phase 1 from cold, plus the versioned schema contract that every other repo depends on. See [`docs/architecture/12-implementation-plan.md`](docs/architecture/12-implementation-plan.md) for the full 11-phase roadmap from here through a real-AWS, Kubernetes-deployed flagship demo.

## The AgentForge repositories

AgentForge is a **polyrepo**: seven sibling repositories, each with a single clear responsibility, cloned side-by-side under one local workspace folder (see [workspace-setup.md](workspace-setup.md)). This repo is one of the seven — the front door.

| Repository | Purpose |
|---|---|
| `agentforge-docs` | *(this repo)* Architecture documentation and ADRs — the platform's front door. |
| `agentforge-agent-schema` | The versioned JSON Schema contract for agent YAML, plus example instances. Depended on by the three repos below via pinned git dependencies. |
| `agentforge-frontend` | Next.js + TypeScript + Tailwind + Monaco + React Query developer portal for authoring, deploying, and observing agents. |
| `agentforge-control-plane` | Python + FastAPI + Pydantic service owning agent registry, YAML validation, tool/model/knowledge registries, policy authoring, deployment management, evaluation management, and all database schema migrations. |
| `agentforge-agent-execution-platform` | Python service bundling the LangGraph runtime, Temporal workflows, MCP tool gateway, RAG, memory, model gateway, runtime policy enforcement, and OpenTelemetry instrumentation as one deployable. |
| `agentforge-showcase-agents` | GitOps-style source-of-truth YAML for the demo agents (Company Knowledge Agent, Production Incident Investigator, Fraud Investigation Agent, Supervisor), deployed through the control plane's normal API. |
| `agentforge-infra` | Docker, Kubernetes, Helm, and Terraform for local development and cloud deployment. |

For the full rationale behind this split, see [ADR-0001](docs/adr/0001-polyrepo-vs-monorepo.md) and [architecture doc 06](docs/architecture/06-repository-structure.md).
