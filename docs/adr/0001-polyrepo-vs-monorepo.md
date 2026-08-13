# 0001. Polyrepo over Monorepo

## Status

Accepted

## Context

AgentForge is composed of seven distinct pieces of work: architecture documentation, a versioned schema contract, a Next.js frontend, a Python control plane, a Python execution platform, GitOps-style demo agent YAML, and infrastructure-as-code. These pieces have different languages/toolchains (TypeScript+pnpm, Python+uv, JSON Schema, Terraform/Helm/YAML, Markdown), different release cadences, and — most importantly — one genuine cross-cutting *contract* (the agent YAML schema) that needs to be consumed identically by three otherwise-independent components. We needed to decide whether all of this lives in one repository or is split across several.

## Decision

Split AgentForge into seven sibling repositories — `agentforge-docs`, `agentforge-agent-schema`, `agentforge-frontend`, `agentforge-control-plane`, `agentforge-agent-execution-platform`, `agentforge-showcase-agents`, `agentforge-infra` — cloned side by side under one local workspace folder, each with its own git history and (eventually) its own CI and remote. `agentforge-agent-schema` is consumed by the three dependent repos via pinned git dependencies, not copy-pasted or inlined.

## Alternatives Considered

- **Single monorepo** with all seven pieces as subdirectories, using a build tool (e.g. Nx, Turborepo, Bazel) for cross-cutting concerns. Rejected: it would still need internal versioning discipline for the schema contract to give the same "this is a real interface" guarantee polyrepo gives for free via the filesystem/git boundary, and it would mix languages/toolchains in one repo for no benefit this project's scale actually needs — a monorepo's main payoff (atomic cross-cutting commits, shared tooling) matters most at a scale and team size AgentForge doesn't have.
- **Two repos** (one "platform" repo for control+execution plane, one "everything else" repo). Rejected: this collapses exactly the plane boundary ([ADR-0002](0002-control-execution-plane-separation.md)) the architecture is built around, and still leaves the schema-contract problem unsolved (would the schema live in the platform repo, coupling the frontend to it as a cross-repo dependency anyway, or in the "everything else" repo, adding an odd dependency direction from platform code into a grab-bag repo).

## Consequences

**Positive:** each repo has an unambiguous single responsibility and toolchain; the schema contract is versioned and consumed like a real published dependency, forcing schema changes to be deliberate and visible; the polyrepo boundary itself becomes the enforcement mechanism for the control/execution plane split at the data layer (see [ADR-0006](0006-timescaledb-single-database.md)), since each plane's code genuinely cannot import the other's internals — only call its API.

**Negative / failure modes:** cross-repo changes (e.g., a schema field addition needed by both the frontend and the control plane) require sequencing — tag the schema repo first, then bump the pinned dependency in each consumer — rather than one atomic commit. This is slower than a monorepo's cross-cutting commit and is an accepted cost for a solo-maintained project; it would become a more meaningful cost with multiple concurrent contributors making frequent cross-cutting changes (tracked as a note in [doc 13](../architecture/13-risks.md)).

**Operational:** `agentforge-infra`'s local Docker Compose setup depends on all repos being cloned as git siblings (see [workspace-setup.md](../../workspace-setup.md)) — a real, if low-cost-to-fix, constraint this decision introduces.

**Scaling:** polyrepo scales well to more contributors and more independent release cadences; it does not, by itself, provide anything a monorepo build tool wouldn't also provide for dependency graph awareness — the benefit here is specifically about legible, enforceable boundaries, not build performance.
