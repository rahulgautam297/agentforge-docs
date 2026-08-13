# Workspace Setup

AgentForge is split across **seven sibling git repositories** rather than one monorepo (see [ADR-0001](docs/adr/0001-polyrepo-vs-monorepo.md)). None of them depend on the others being nested inside them — they depend on being laid out as *siblings* in one local folder, because `agentforge-infra`'s `docker compose` files use relative build contexts (e.g. `../agentforge-frontend`, `../agentforge-control-plane`) to build every service from source in local development.

## Required layout

Clone or create all seven repositories under one workspace folder — this documentation assumes `~/dev/agentforge/` (any path works, but relative paths inside `agentforge-infra` assume this sibling structure):

```
~/dev/agentforge/
├── agentforge-docs/                      # architecture docs + ADRs (this repo)
├── agentforge-agent-schema/              # versioned JSON Schema contract for agent YAML
├── agentforge-frontend/                  # Next.js developer portal
├── agentforge-control-plane/             # FastAPI control plane
├── agentforge-agent-execution-platform/  # LangGraph + Temporal + tools + RAG + memory runtime
├── agentforge-showcase-agents/           # demo agent YAML (GitOps source of truth)
└── agentforge-infra/                     # docker-compose, k8s, helm, terraform
```

The workspace folder itself (`~/dev/agentforge/`) is **not** a git repository — it is just a container directory. Each of the seven subdirectories is its own independent git repository with its own history and its own remote on GitHub (CI is still unwired — see Phase 11).

## Repository purposes

| Repository | Language / stack | Owns |
|---|---|---|
| `agentforge-docs` | Markdown | Architecture documentation, ADRs — the front door for anyone new to the project. |
| `agentforge-agent-schema` | JSON Schema | The single source of truth for what a valid agent YAML document looks like, plus example instances. Plane-neutral: it has no opinion about control plane or execution plane internals, only about the contract between everything that produces or consumes agent YAML. |
| `agentforge-frontend` | TypeScript, Next.js, Tailwind, Monaco, React Query | The developer-facing web UI: agent authoring (YAML editor + live preview), deployment, execution/trace viewing, approvals, tool/model/knowledge/evaluation management. |
| `agentforge-control-plane` | Python, FastAPI, Pydantic | Agent registry, YAML validation (semantic layer), model/tool/knowledge registries, policy authoring, deployment manager, evaluation manager. The sole owner of database schema migrations. |
| `agentforge-agent-execution-platform` | Python | The LangGraph runtime, Temporal workflows, MCP tool gateway, RAG pipeline, memory services, model gateway, runtime policy enforcement, and OpenTelemetry instrumentation — bundled as internal modules of one deployable service, not split into further repos. |
| `agentforge-showcase-agents` | YAML | The declarative source of truth for the four demonstration agents. Edited here, deployed exclusively through the control plane's normal validate → deploy API; never read directly by the execution platform at runtime. |
| `agentforge-infra` | Docker Compose, Kubernetes manifests, Helm charts, Terraform | Everything needed to run the full stack locally (`docker compose up`) or deploy it to a cluster. |

## Why sibling directories, specifically

`agentforge-infra`'s local `docker-compose.yml` builds several services from source (rather than pulling prebuilt images) so that a developer's uncommitted local changes in, say, `agentforge-control-plane` are picked up on the next `docker compose up --build`. Docker Compose build contexts are resolved relative to the location of the compose file, so a build context of `../agentforge-control-plane` only resolves correctly if `agentforge-infra` and `agentforge-control-plane` are siblings on disk. Cloning the repositories into arbitrary, unrelated locations will break local `docker compose` builds; there is currently no environment variable or `.env` override for these paths (a possible Phase-1 hardening item is documented in [architecture doc 13](docs/architecture/13-risks.md)).

## Current state (Phase 0)

All seven repositories are hosted publicly on GitHub under the `rahulgautam297` account, one repo per row above. To set up the workspace from scratch on another machine:

```
mkdir -p ~/dev/agentforge && cd ~/dev/agentforge
gh repo clone rahulgautam297/agentforge-docs
gh repo clone rahulgautam297/agentforge-agent-schema
gh repo clone rahulgautam297/agentforge-frontend
gh repo clone rahulgautam297/agentforge-control-plane
gh repo clone rahulgautam297/agentforge-agent-execution-platform
gh repo clone rahulgautam297/agentforge-showcase-agents
gh repo clone rahulgautam297/agentforge-infra
```

(or the equivalent `git clone https://github.com/rahulgautam297/<repo>.git` for each, if you don't have `gh` installed). CI/CD (GitHub Actions, ArgoCD-ready deployment) is still unwired — that's addressed in Phase 11 of the [implementation plan](docs/architecture/12-implementation-plan.md); for now these are plain hosted repos with no workflows configured.

## Package managers

- **Frontend** (`agentforge-frontend`): [pnpm](https://pnpm.io/).
- **Python repos** (`agentforge-control-plane`, `agentforge-agent-execution-platform`): [uv](https://docs.astral.sh/uv/).

Both are fast, lockfile-driven package managers chosen to keep local setup reproducible and quick — see each repo's own README (added when that repo is populated in its respective phase) for exact setup commands.
