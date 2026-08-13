# Architecture Decision Records

Lightweight [MADR](https://adr.github.io/madr/)-style records: each documents a decision's context, the decision itself, alternatives considered (including rejected ones — including a rejected earlier draft, in two cases), and consequences (failure modes, unavailability, consistency, scaling, where relevant). See [docs/architecture/README.md](../architecture/README.md#which-adrs-back-which-decisions) for which architecture document each ADR backs.

| # | Title | Status | One-line decision |
|---|---|---|---|
| [0001](0001-polyrepo-vs-monorepo.md) | Polyrepo over Monorepo | Accepted | Seven sibling repositories, not one monorepo — the schema contract is consumed like a real versioned dependency, and each piece keeps its own toolchain. |
| [0002](0002-control-execution-plane-separation.md) | Control Plane / Execution Plane Separation | Accepted | Two independently deployable planes — agent *definitions* vs. agent *behavior* — so control-plane downtime never stops an in-flight execution. |
| [0003](0003-langgraph-for-agent-orchestration.md) | LangGraph for Agent Orchestration | Accepted | LangGraph owns the reasoning graph — nodes, conditional edges, in-execution state, multi-agent handoff — and nothing about durability. |
| [0004](0004-temporal-for-durable-execution.md) | Temporal for Durable Execution | Accepted | Temporal wraps the LangGraph invocation as a unit (never every individual call) to provide retries, timers, crash recovery, and human-approval waits. |
| [0005](0005-bedrock-primary-with-provider-abstraction.md) | Bedrock Primary, with Provider Abstraction | Accepted | Bedrock is the target production provider, but every model call goes through a Model Gateway so `mock`/`openai` are equally first-class. |
| [0006](0006-timescaledb-single-database.md) | Single TimescaleDB Database (Not Split by Plane) | Accepted | One shared `agentforge` database with role-based access control enforcing the plane boundary — **supersedes an earlier rejected two-database draft.** `execution_steps` is a Timescale hypertable. |
| [0007](0007-opensearch-hybrid-retrieval.md) | OpenSearch for Hybrid Retrieval | Accepted | OpenSearch provides combined lexical + vector retrieval for all knowledge bases, run locally as a single-node container. |
| [0008](0008-mcp-tool-abstraction-and-policy-engine.md) | MCP Tool Abstraction and Policy Engine | Accepted | Every tool call speaks MCP through one gateway choke point, gated by a declarative Policy Engine returning ALLOW/DENY/HUMAN_APPROVAL. |
| [0009](0009-human-approval-mechanism.md) | Human Approval Mechanism | Accepted | Human approval is a Temporal signal-wait — zero cost while pending, durable across restarts, auto-resolves on timeout. |
| [0010](0010-modular-monolith-within-each-repo.md) | Modular Monolith Within Each Repo | Accepted | Each plane is one deployable with internal modules, not a microservice per capability — split only where isolation is actually earned. |

## Notes on the two "superseding" ADRs

[ADR-0002](0002-control-execution-plane-separation.md) and [ADR-0006](0006-timescaledb-single-database.md) both explicitly document a rejected earlier draft: an initial plan proposed two physically separate Postgres databases, one per plane, to enforce the control/execution plane boundary at the data layer. That draft was reconsidered and rejected during planning — the polyrepo split already provides real plane separation (separate codebases, CI, and deploys), so a second physical database was redundant operational cost, and it would have destroyed real foreign-key relationships across the plane boundary (`executions.deployment_id`, `executions.agent_version_id`) in favor of value-only cross-database references. The final design uses one shared database (`agentforge`) with the plane boundary enforced by Postgres role grants instead — full reasoning trail in ADR-0006.
