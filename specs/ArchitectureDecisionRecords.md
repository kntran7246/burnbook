# Architecture Decision Records

## ADR-001: Quote collection membership

**Status:** Proposed for v1

**Decision:** A quote belongs to exactly one collection in v1.

**Rationale:** This simplifies collection permissions, search filtering, moderation, and Discord channel mapping.

**Future option:** Add a `quote_collection` join table if cross-collection quote reuse becomes necessary.

**Consequences:** Moving a quote between collections is an explicit operation and must be permission-checked and audit-logged.

## ADR-002: Personal tierlists

**Status:** Proposed for v1

**Decision:** Tierlist rankings are per user and per collection.

**Rationale:** Users can rank quotes independently without write conflicts over a shared ranking.

**Consequences:** Aggregate rankings, if added, are derived views rather than the source of truth.

## ADR-003: Explicit speaker relinking

**Status:** Proposed for v1

**Decision:** Unlinked speaker names are not automatically matched to collection speakers. Relinking is an explicit moderator action.

**Rationale:** Name matching can incorrectly merge distinct people and silently change historical data.

**Consequences:** Relinking creates a quote revision and an audit event.

## ADR-004: Discord ingestion direction

**Status:** Proposed for v1

**Decision:** The bot ingests quotes from Discord; application-to-Discord posting is deferred.

**Rationale:** One-way ingestion reduces synchronization conflicts and operational complexity for the first release.

**Consequences:** Discord events require idempotency keys and retry handling. Outbound publishing can be added later.

## ADR-005: PostgreSQL search

**Status:** Proposed for v1

**Decision:** Use PostgreSQL full-text search with `tsvector` and `pg_trgm`.

**Rationale:** It avoids an additional operational dependency while the dataset and search requirements are still modest.

**Consequences:** Search schema and query performance must be load-tested before launch. Elasticsearch/OpenSearch remains a future migration option.
