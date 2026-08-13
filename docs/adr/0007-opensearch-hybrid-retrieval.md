# 0007. OpenSearch for Hybrid Retrieval

## Status

Accepted

## Context

The Company Knowledge Agent and other RAG-capable agents (notably the Incident Investigator, retrieving from `engineering-docs` and `incident-history` knowledge bases) need retrieval over ingested documents that combines keyword/lexical matching (good for exact terms — error codes, service names, specific identifiers) with semantic/vector similarity (good for conceptually related but lexically different phrasing). We needed a retrieval backend supporting both, indexable and queryable locally with no cloud dependency (per [doc 11](../architecture/11-local-development.md)'s zero-AWS-credential goal through Phase 9), and operable by a solo engineer without a large new ops surface.

## Decision

Use **OpenSearch** for both ingestion-time indexing and query-time hybrid retrieval (combining lexical/BM25 search with vector similarity search) for all knowledge bases. Ingestion (chunking, embedding, indexing) and retrieval both live in the execution plane's RAG Pipeline module (see [doc 04](../architecture/04-execution-plane.md)). Locally, a single-node OpenSearch container is used (see [doc 11](../architecture/11-local-development.md)); production topology is addressed in Phase 11.

## Alternatives Considered

- **A dedicated vector-only database** (e.g. a standalone vector DB with no native lexical search). Rejected: would require a second system for keyword matching or would give up hybrid retrieval quality, which matters directly for the Incident Investigator's need to match specific error codes/service names as well as conceptually related incident history.
- **pgvector inside the shared `agentforge` TimescaleDB.** Considered, since it would avoid introducing a new technology at all. Rejected for this project because OpenSearch's hybrid (lexical + vector) query support and purpose-built full-text search are considerably more mature than combining pgvector with Postgres full-text search for this use case, and knowledge-base documents are a genuinely different data shape (unstructured text corpora, not structured relational rows) from everything else in `agentforge` — conflating them into the same database would blur a distinction ([ADR-0006](0006-timescaledb-single-database.md)) that's kept intentionally clean for the actual relational data.
- **A managed cloud search service** (e.g. a hosted vector search offering). Rejected for the same reason as most other cloud-dependency choices in this project: it would break the zero-AWS-credential local development goal through Phase 9.

## Consequences

**Positive:** hybrid retrieval quality (lexical + semantic) out of the box; fully local-runnable with no cloud dependency; mature, well-documented technology with a large operational knowledge base.

**Negative / failure modes:** OpenSearch is a genuinely new piece of operational surface beyond the Postgres-family stack used everywhere else — a second thing to run, monitor, and back up. If OpenSearch is unavailable, RAG-dependent agents fail retrieval (surfaced as a normal execution error, or triggering the agent's `retry_policy` in `durable` mode) but this does not affect non-RAG agents or any control-plane function.

**State persisted:** ingested/indexed document content and vector embeddings live in OpenSearch, distinct from the `documents` table in `agentforge` (which holds document *metadata* — `source_uri`, `checksum`, `indexed_at`, `chunk_count` — not the indexed content itself). This is a deliberate split: `agentforge` is the metadata/registry source of truth (what documents exist, when were they last indexed), OpenSearch is the retrieval index.

**Consistency model:** eventually consistent between ingestion and searchability, as is standard for search-index technology — a newly ingested document is queryable once OpenSearch's own indexing completes, not instantaneously on `documents` row insert; this window is not currently surfaced to the frontend as an explicit "indexing in progress" state beyond the `knowledge_bases.status` / `documents.indexed_at` fields already in the schema (see [doc 08](../architecture/08-database-schema.md)).

**Scaling:** single-node locally; production clustering (sharding/replication) is a Phase 11 concern, addressed alongside the rest of the production infrastructure work.
