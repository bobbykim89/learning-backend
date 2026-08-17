# Capstone Projects

Build these in order. Each one forces the concepts from its phases into your hands.

Project **code lives in a separate repository per project**. The files linked here
hold notes and design decisions only — what was decided, what went wrong, what the
measured result was.

Files are created when the project is under way — see [CLAUDE.md](../../CLAUDE.md).

1. [URL shortener with real constraints](url-shortener.md) — ID generation, caching, rate limiting, analytics via async writes. _(Phases 0, 2, 4)_
2. [Multi-tenant task API](multi-tenant-task-api.md) — layered architecture with DI, RBAC, RLS, cursor pagination, OpenAPI, full test suite. _(Phases 1, 3, 5, 9)_
3. [Payments-style ledger](payments-ledger.md) — double-entry accounting, transactional integrity, idempotency keys, exactly-once webhooks, reconciliation. _(Phases 2, 5, 6)_
4. [Event-driven order pipeline](event-driven-order-pipeline.md) — outbox pattern, message broker, saga with compensation, DLQ, replay tooling. _(Phase 6)_
5. [High-throughput read service](high-throughput-read-service.md) — multi-layer caching, stampede protection, load tested to a documented p99 target with tracing and dashboards. _(Phases 4, 7)_
6. [Ship one of the above properly](ship-it-properly.md) — containerized, IaC-provisioned, CI/CD with zero-downtime migrations, SLOs, alerts, and a tested restore. _(Phases 7, 8)_
