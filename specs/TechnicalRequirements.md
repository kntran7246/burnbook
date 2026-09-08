# Quote Platform — Technical Requirements
**Version:** 0.1 (Draft)
**Last updated:** 2026-09-07
**Status:** Pre-architecture review

## 1. Performance and Capacity

- Support 1,000 simultaneously authenticated users.
- Sustain 100 requests per second for 15 minutes and tolerate bursts of 250 requests per second.
- Target API p95 latency below 500 ms for reads and below 1 second for writes under the sustained-load target.
- Define load-test scenarios for quote browsing, search, quote creation, reactions, and tierlist updates before launch.

## 2. Availability and Recovery

- Target 99.5% monthly availability for v1.
- Define a backup schedule and test database restoration before production launch.
- Target an RPO of 15 minutes and an RTO of 1 hour unless deployment constraints require different targets.
- Every service must expose liveness and readiness health checks.

## 3. API and Error Handling

- Publish an OpenAPI specification as part of the application source.
- Version externally consumed APIs.
- Use RFC 7807 Problem Details for errors.
- Validation errors must include a stable machine-readable error code and field-level details where applicable.
- Use pagination consistently across collection, quote, search, report, and audit endpoints.
- Use UTC ISO 8601 timestamps in API responses.
- Write endpoints that may be retried must define idempotency behavior.

## 4. Security and Access

- Effective authorization is the intersection of API-key scope, current user permissions, and resource visibility.
- Store API-key hashes, not plaintext keys; support creation, rotation, revocation, and last-used timestamps.
- OAuth provider identifiers must be unique and must not be used as public user identifiers.
- Protect browser sessions against CSRF and enforce an explicit CORS policy.
- Rate-limit API keys, authenticated sessions, and Discord commands independently.
- Return `429 Too Many Requests` with `Retry-After` when a limit is exceeded.
- Log authentication, authorization, role changes, API-key changes, moderation actions, and data exports.
- Do not log access tokens, API keys, or other secrets.

## 5. Data and Migrations

- PostgreSQL is the system of record.
- Liquibase owns schema changes; every migration must be reviewed and run in CI against a clean database.
- Repository integration tests must use Testcontainers with real PostgreSQL.
- Soft-deleted content must be excluded from normal reads and search while remaining available to authorized moderation/audit workflows.
- Define backup retention, deletion retention, and account-deletion behavior before production.

## 6. Search

- Use PostgreSQL full-text search with `tsvector` and `pg_trgm` for v1.
- Search must cover quote text and speaker names.
- Search results must enforce collection visibility, roles, soft deletion, and blocking rules.
- Elasticsearch/OpenSearch is explicitly deferred until scale or relevance requirements justify it.

## 7. Testing and Delivery

- Unit-test service and business logic.
- Integration-test repositories and Liquibase migrations against real PostgreSQL.
- Validate API contract tests against the OpenAPI specification.
- Test Discord parsing and command handling against mocked gateway/REST interfaces.
- Run all test tiers on every pull request.
- Enforce coverage thresholds for quotes, collections, authorization, and moderation modules.
- Run dependency, static-analysis, and container-image security checks in CI.

## 8. Observability

- Export application and infrastructure metrics in Prometheus format.
- Provide Grafana dashboards for traffic, latency, errors, database health, sync failures, and rate limits.
- Include correlation IDs in logs and API responses where appropriate.
- Use structured logs and redact secrets and personal data.
- Add distributed tracing when the application has more than one production service or asynchronous integration.

## 9. Infrastructure

- Package services as Docker images.
- Keep local development runnable with Docker Compose.
- Select the production orchestrator after v1 load and operational requirements are validated; Kubernetes is not mandatory for the first deployment.
- Infrastructure must be reproducible through Terraform where infrastructure-as-code is used.

## 10. Client and Accessibility

- Use React with shadcn components and TanStack Router.
- Support the latest two versions of major desktop browsers and a defined mobile browser baseline.
- Target WCAG 2.1 AA for primary workflows.
- The client must display field-level validation errors and actionable API errors.
- Primary workflows must work at mobile, tablet, and desktop viewport widths without horizontal scrolling.
- Touch targets must be appropriately sized and spaced, and essential actions must not rely on hover-only interactions.
- Test responsive layouts on a representative narrow phone viewport, tablet viewport, and desktop viewport.
- Support keyboard navigation, visible focus states, semantic landmarks, accessible names, and screen-reader-compatible form feedback.
