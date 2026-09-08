# Quote Platform — Technology Stack

**Version:** 0.1 (Draft)
**Last updated:** 2026-09-07
**Status:** Pre-architecture review

## Frontend

- React
- shadcn/ui
- TanStack Router

## Backend

- Java
- Spring Boot
- OpenAPI

## Data and integrations

- PostgreSQL — primary database and v1 search
- Liquibase — database migrations
- Redis — rate limiting, caching, and shared session/state where needed
- Discord API — bot integration

## Quality and operations

- JUnit and the Spring test stack — unit and application tests
- Testcontainers — repository and migration integration tests
- Docker — local and deployment containers
- Docker Compose — local development
- Prometheus — metrics
- Grafana — dashboards
- OpenTelemetry — tracing when multi-service tracing is required
- Terraform — infrastructure as code where applicable

## Deferred or conditional

- Elasticsearch/OpenSearch — only if PostgreSQL search no longer meets scale or relevance needs
- Kubernetes — introduced when operational or capacity requirements justify it
- Ansible — use only where host configuration management is required
