# 0005. Bedrock as Primary Provider, Behind a Provider Abstraction

## Status

Accepted

## Context

AgentForge's flagship demo and enterprise-platform framing are explicitly inspired by internal AI platforms at large fintechs, where AWS Bedrock is a common, compliance-friendly way to access foundation models without sending data to a third-party API directly. At the same time, local development (see [doc 11](../architecture/11-local-development.md)) needs to work with zero AWS credentials through Phase 9, and the platform should not be hard-locked to one LLM vendor at the code level, since agent YAML (`model.provider`) is meant to be a genuine choice, not a fixed constant.

## Decision

Treat `bedrock` as the primary/target production provider, but implement all model access behind a **Model Gateway** abstraction in the execution plane, with `provider` (`bedrock` | `mock` | `openai`) as a first-class enum in the agent schema (see [doc 07](../architecture/07-agent-yaml-schema.md)). Real Bedrock integration is Phase 10 of the roadmap; `MockProvider` is the default for all local development and Phases 0–9, giving deterministic canned responses with simulated latency. `openai` exists in the schema/gateway as a second real-provider option, demonstrating the abstraction isn't Bedrock-specific in practice, not just in principle.

## Alternatives Considered

- **Direct Bedrock SDK calls scattered through the LangGraph node code, no gateway abstraction.** Rejected: would tightly couple the reasoning graph to one provider's request/response shape, make `MockProvider` substitution (required for local dev) awkward, and make adding `openai` later a graph-code change rather than a gateway-config change.
- **OpenAI as primary, Bedrock as secondary.** Rejected: doesn't match the enterprise-fintech-platform framing this project is inspired by, where Bedrock's compliance posture (data residency, no third-party data sharing) is typically the reason an internal platform standardizes on it.
- **No mock provider — require real credentials even locally.** Rejected outright: contradicts the explicit goal (see [doc 11](../architecture/11-local-development.md)) of zero-AWS-credential local development through Phase 9, which is essential to this project being runnable and demoable without cloud setup.

## Consequences

**Positive:** agent YAML is portable across providers with no code change; local development and CI never need real credentials or incur real API cost through Phase 9; adding a new provider is a Model Gateway change, isolated from LangGraph graph code and from every existing agent's YAML.

**Negative / failure modes:** the abstraction has to account for provider-specific differences (rate limits, error shapes, streaming behavior, token-counting nuances) generically enough that no agent-specific branching leaks in — a real design discipline cost, paid once in the gateway rather than repeatedly per agent.

**What happens when a provider is unavailable:** a model call failure surfaces as a normal execution error (or, for `durable` mode, triggers the agent's declared `retry_policy` before surfacing as an error) — the gateway does not silently fail over to a different provider than the one the agent's YAML declared, since that would violate the principle that an agent's YAML is the authoritative description of what it does.

**Scaling:** the Model Gateway is a stateless routing layer; it imposes no scaling bottleneck of its own beyond whatever limits the underlying provider (Bedrock, OpenAI, or the in-process `MockProvider`) imposes.
