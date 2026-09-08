# Quote Platform — Product Requirements

**Version:** 0.1 (Draft)
**Last updated:** 2026-09-07
**Status:** Draft — product decisions pending

---

## 1. Overview

A web application for creating, organizing, ranking, and reacting to "quotes" tied to collections and the people who said them, with a companion Discord bot for capturing and syncing activity from Discord servers.

---

## 2. Product Goals

The application helps users capture memorable quotes, organize them into collections, rank and react to them, and review how quotes change over time. A companion Discord bot allows quotes and activity to be captured from Discord.

The v1 product prioritizes reliable quote and collection workflows, clear collection-level permissions, and a simple Discord ingestion path.

## 3. Feature Requirements

### 3.1 Quotes & Collections
- Users can create quotes, each associated with a collection.
- A quote belongs to exactly one collection in v1. See ADR-001 for the migration path to multiple collections.
- Quotes can be associated with a speaker in one of two ways:
  - A person already added to the collection (referenced by a display/fake name)
  - An individual not yet added to the collection (unlinked/free-text speaker)
- Speaker identity is modeled as a first-class entity. A quote may reference a collection speaker or retain an unlinked speaker name.
- Relinking an unlinked speaker is an explicit moderator action and must create quote revisions; it is never performed implicitly by name matching.

### 3.2 Tierlists
- Users can rank quotes within a collection using a tierlist.
- Tierlists are personal per-user rankings in v1. A collection may expose an optional aggregate view, but the source of truth is each user's ranking.

### 3.3 Reactions
- Users can react to quotes.
- Reactions use a fixed, application-defined set in v1. Users may have at most one reaction of each type per quote.

### 3.4 Quote Changelogs
- All quote edits must produce a changelog entry.
- Changelogs should support diffs (what changed, not just that something changed), not just a generic audit log line.
- **Design recommendation:** implement as a `quote_revision` table (event-sourcing-lite) rather than a generic audit log.

### 3.5 Users, Friends & Blocking
- Users can send/accept friend requests.
- Users can block other users.
- Blocking prevents direct interaction and hides the blocked user's content and reactions from the blocker. Existing collection permissions still govern content visible to other users.
- Friendship does not grant implicit access to private collections. Private collection access is explicit and represented as a collection membership/role.

### 3.6 Collection Visibility
- Collections can be marked public or private.
- **Future consideration (not required for v1):** quote-level visibility overrides within a collection, for cases where a private collection still needs to hide a specific sensitive quote.

### 3.7 Discord Integration
- A dedicated Discord bot connects Discord servers to the web application.
- The v1 bot ingests quotes from Discord into the application. Posting application activity back to Discord is deferred.
- One bot installation may serve multiple guilds. A guild may map to multiple collections, with the source channel selecting the collection.
- Bot must respect Discord's own outbound rate limits, independent of the application's internal rate limiting (see 3.3).

### 3.8 Authentication
- Account creation via OAuth.
- Discord OAuth is recommended as the primary provider, given the existing bot integration.

### 3.9 Accessibility and Responsive UI
- Primary workflows must target WCAG 2.1 AA.
- The application must be usable on mobile phones, tablets, and desktop screens without requiring horizontal scrolling for primary workflows.
- Quote browsing, quote creation, search, reactions, tierlist ranking, authentication, and moderation must be usable with touch, keyboard, and supported assistive technologies.
- Responsive layouts must preserve access to essential actions at small viewport widths; actions must not depend on hover alone.

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
- API keys have explicit scopes, but effective access is the intersection of the key scope, the issuing user's current permissions, and resource visibility. A key cannot grant access the user does not have.
- **Design recommendation:** model roles and permissions as database-backed tables (`role`, `permission`), not hardcoded enums, so permission sets can change without a redeploy.

### 3.2 Search
- Full-text search over quotes.
- **Design recommendation:** start with Postgres `tsvector`/`pg_trgm`; avoid introducing Elasticsearch/OpenSearch until there's a concrete scaling need.
- Search is available within a collection and across collections the user can access. Public-only global search is deferred.
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
- Must support 1,000 simultaneously authenticated users, 100 sustained requests per second for 15 minutes, and short bursts of 250 requests per second.

### 4.2 API & Error Handling
- Expose an OpenAPI specification; support application-issued API keys (see 3.1 for scoping).
- Errors must propagate to the frontend with actionable, field-level feedback.
- Use RFC 7807 Problem Details for errors. Validation errors must include machine-readable codes and field-level details.

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
- React with shadcn as the primary component library and TanStack Router for routing.

### 4.7 Infrastructure
- Containerization via Docker.
- Local development must work with Docker Compose.
- Production orchestration remains conditional; Kubernetes is introduced only when load or operational requirements justify it.

### 4.8 Real-Time Updates
- Real-time delivery is out of scope for v1. The client uses normal API reads and writes; the transport can be added later without changing the domain model.

---

## 5. Remaining Product Decisions

| # | Topic | Decision Needed |
|---|---|---|
| 1 | Infrastructure | Full Kubernetes at launch, or a simpler deployment first? |
| 2 | Moderation | What actions can reviewers take, and what is the appeal/retention policy? |
| 3 | Discord parsing | Which message format or command creates a quote, and which channels are allowed? |
| 4 | Tierlist presentation | What tier names, ordering, and aggregate views should the UI provide? |

---

## 6. Out of Scope (v1)

- Quote-level visibility overrides within otherwise public/private collections
- Elasticsearch/OpenSearch-based search
- Multi-provider OAuth beyond the initial provider(s) selected

---

*Next steps: resolve the remaining decisions in Section 5, review the ADRs, then proceed to the ERD and system architecture design.*
