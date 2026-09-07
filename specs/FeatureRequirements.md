# Quote Platform — Requirements Document

**Version:** 0.1 (Draft)
**Last updated:** 2026-09-07
**Status:** Pre-architecture review

---

## 1. Overview

A web application for creating, organizing, ranking, and reacting to "quotes" tied to collections and the people who said them, with a companion Discord bot for capturing and syncing activity from Discord servers.

---

## 2. Feature Requirements

### 2.1 Quotes & Collections
- Users can create quotes, each associated with a collection.
- **Open decision:** can a quote belong to multiple collections, or exactly one?
- Quotes can be associated with a speaker in one of two ways:
  - A person already added to the collection (referenced by a display/fake name)
  - An individual not yet added to the collection (unlinked/free-text speaker)
- **Design recommendation:** model "speaker" as its own polymorphic entity (`LINKED` vs `UNLINKED`) rather than scattering nullable foreign keys across the schema.
- **Open decision:** if an unlinked speaker is later added to the collection, do historical quotes retroactively relink to them?

### 2.2 Tierlists
- Users can rank quotes within a collection using a tierlist.
- **Open decision:** is the tierlist a single shared/global ranking per collection, or does each user maintain their own personal ranking of the same quote pool? This materially changes the data model and concurrency handling.

### 2.3 Reactions
- Users can react to quotes.
- **Open decision:** fixed set of reaction types (enum) vs. open-ended/custom reactions (affects schema and Discord sync).

### 2.4 Quote Changelogs
- All quote edits must produce a changelog entry.
- Changelogs should support diffs (what changed, not just that something changed), not just a generic audit log line.
- **Design recommendation:** implement as a `quote_revision` table (event-sourcing-lite) rather than a generic audit log.

### 2.5 Users, Friends & Blocking
- Users can send/accept friend requests.
- Users can block other users.
- **Open decision:** does blocking hide a blocked user's quotes/reactions, prevent interaction only, or both?
- **Open decision:** should friendship grant elevated visibility into private collections (e.g., an implicit Viewer role for friends of members)?

### 2.6 Collection Visibility
- Collections can be marked public or private.
- **Future consideration (not required for v1):** quote-level visibility overrides within a collection, for cases where a private collection still needs to hide a specific sensitive quote.

### 2.7 Discord Integration
- A dedicated Discord bot connects Discord servers to the web application.
- **Open decision:** bot direction of sync — does it only ingest quotes from Discord into the app, only post app activity back to Discord, or both (bidirectional)?
- **Open decision:** does one bot instance serve multiple Discord guilds, and how does a guild map to a collection (1:1, many:1)?
- Bot must respect Discord's own outbound rate limits, independent of the application's internal rate limiting (see 3.3).

### 2.8 Authentication
- Account creation via OAuth.
- Discord OAuth is recommended as the primary provider, given the existing bot integration.

---

## 3. Access Control & Platform Requirements

### 3.1 Role-Based Access Control (RBAC)
- Roles are scoped **per collection**, not only globally — a user may hold different roles across different collections.
- Baseline collection roles:

| Role | Permissions |
|---|---|
| Owner | Full control — delete collection, manage roles, transfer ownership |
| Moderator | Edit/delete quotes, resolve reports, manage members |
| Member | Add quotes, react, rank |
| Viewer | Read-only (public collections, or private collections shared with friends) |

- Separate **global/platform roles** (e.g., Admin, Support) exist for cross-collection moderation.
- API keys carry independent scopes (read-only vs. write) that are not directly tied to the issuing user's role, so a leaked key cannot grant more access than it was scoped for.
- **Design recommendation:** model roles and permissions as database-backed tables (`role`, `permission`), not hardcoded enums, so permission sets can change without a redeploy.

### 3.2 Search
- Full-text search over quotes.
- **Design recommendation:** start with Postgres `tsvector`/`pg_trgm`; avoid introducing Elasticsearch/OpenSearch until there's a concrete scaling need.
- **Open decision:** search scope — within a single collection, across all collections a user has access to, or global search limited to public collections.
- Search results must respect RBAC and blocking rules (no surfacing private-collection content or blocked users' content).

### 3.3 Rate Limiting
- Required at multiple layers:
  - Per API key (external/API consumers)
  - Per user session (web app abuse prevention)
  - Per Discord bot command (independent from web app limits, and mindful of Discord's own outbound rate limits)
- **Design recommendation:** token-bucket or sliding-window algorithm backed by Redis, since in-memory rate limiting breaks down across multiple Spring Boot replicas.
- Rate-limit violations should return `429 Too Many Requests` with a `Retry-After` header, consistent with the application's overall error-handling standard (see 4.2).

### 3.4 Testing
- Required baseline test coverage before the codebase grows significantly:
  - **Unit tests** — service and business logic layer
  - **Integration tests** — repository layer using Testcontainers against real Postgres (not mocks), given the Liquibase-managed schema
  - **Contract tests** — validated against the OpenAPI spec to prevent spec/implementation drift
  - **Bot-specific tests** — Discord command handling with a mocked gateway/REST layer
- CI must run all test tiers on every pull request.
- Consider enforcing coverage thresholds on core modules (quotes, collections, permissions) rather than a uniform repo-wide threshold.

### 3.5 Moderation
- Reporting flow for quotes (and potentially users): who can report, who reviews, and what actions reviewers can take (hide, edit, delete, ban).
- Moderation actions must themselves be audit-logged (who acted, what action, when, why) — consistent with the quote changelog philosophy.
- Moderation queues are scoped by role: collection Moderators see reports within their own collection; platform Admins see everything.
- **Design recommendation:** soft-delete ("shadow" state) for removed content rather than hard delete, to preserve audit trails and support appeals.

---

## 4. Technical Requirements

### 4.1 Scale
- Must support at least 1,000 concurrent users.
- **Open decision:** clarify whether "concurrent users" means concurrent live connections (WebSocket/API) or a sustained requests/sec target — this determines caching and read-replica strategy up front.

### 4.2 API & Error Handling
- Expose an OpenAPI specification; support application-issued API keys (see 3.1 for scoping).
- Errors must propagate to the frontend with actionable, field-level feedback.
- **Design recommendation:** adopt RFC 7807 (Problem Details) as the standard error response shape; Spring Boot 6+ supports this natively via `ProblemDetail`.

### 4.3 Data Layer
- PostgreSQL as the primary database.
- Liquibase for schema/database definition management.
- Redis for caching, rate limiting, and session/state sharing across replicas.

### 4.4 Backend
- Java with Spring Boot.

### 4.5 Observability
- Prometheus for metrics collection.
- **Design recommendation:** pair with Grafana for dashboards, and consider OpenTelemetry for distributed tracing plus centralized logging (e.g., Loki/ELK) once production debugging becomes routine.

### 4.6 Frontend
- React with shadcn as the primary component library.

### 4.7 Infrastructure
- Containerization via Docker.
- Orchestration via Kubernetes.
- **Consideration:** evaluate whether Docker Compose or a single-node deployment (e.g., Fly.io, ECS) gets the project to launch faster, with Kubernetes introduced later as load justifies it, rather than committing to full K8s for v1.

### 4.8 Real-Time Updates
- Reactions, tierlist changes, and moderation actions are strong candidates for real-time delivery (WebSockets or Server-Sent Events) rather than client polling.
- **Open decision:** scope of real-time features for v1 vs. later iteration.

---

## 5. Open Decisions Summary

| # | Topic | Decision Needed |
|---|---|---|
| 1 | Quotes & collections | Can a quote belong to multiple collections? |
| 2 | Tierlists | Global per-collection ranking, or per-user personal ranking? |
| 3 | Reactions | Fixed enum or open/custom reaction types? |
| 4 | Speaker relinking | Do old quotes retroactively relink when an unlinked speaker joins the collection? |
| 5 | Blocking | Does blocking hide content, restrict interaction, or both? |
| 6 | Friends & visibility | Do friends get implicit Viewer access to private collections? |
| 7 | Discord bot direction | Ingest-only, post-back-only, or bidirectional sync? |
| 8 | Discord guild mapping | Does one bot serve multiple guilds, and how do guilds map to collections? |
| 9 | Concurrency definition | Concurrent connections vs. requests/sec for the 1,000-user target? |
| 10 | Search scope | Per-collection, cross-collection, or public-only global search? |
| 11 | Infrastructure | Full Kubernetes at launch, or simpler deployment first? |
| 12 | Real-time scope | Which features need real-time delivery in v1? |

---

## 6. Out of Scope (v1)

- Quote-level visibility overrides within otherwise public/private collections
- Elasticsearch/OpenSearch-based search
- Multi-provider OAuth beyond the initial provider(s) selected

---

*Next steps: resolve open decisions in Section 5, then proceed to ERD and system architecture design.*
