# Documentation

One file per topic, mirroring the curriculum in the [root README](../README.md).
Progress lives in each topic file's `status:` frontmatter — see
[CLAUDE.md](../CLAUDE.md) for the template and conventions.

A link to a file that does not exist yet means the topic has not been studied.

## Phases

- [Phase 0 — Mental Models](00-mental-models/README.md) — what actually happens when a request arrives
- [Phase 1 — Application Architecture and Code Design](01-architecture/README.md) — where dependency injection, testability, and "how do I structure this" live
- [Phase 2 — Data: Relational Depth First](02-data/README.md) — the single highest-leverage area; most backend performance problems are database problems
- [Phase 3 — Security](03-security/README.md) — a design constraint that shapes every layer, not a checklist to bolt on
- [Phase 4 — Caching and Performance](04-caching-and-performance/README.md) — the cache hierarchy, invalidation, and measuring before optimising
- [Phase 5 — API Design and Contracts](05-api-design/README.md) — resources, pagination, versioning, realtime, and webhooks
- [Phase 6 — Asynchronous and Distributed Systems](06-async-and-distributed/README.md) — queues, delivery semantics, and consistency
- [Phase 7 — Reliability and Operations](07-reliability-and-operations/README.md) — failure handling, observability, and SLOs
- [Phase 8 — Infrastructure and Delivery](08-infrastructure-and-delivery/README.md) — containers, orchestration, IaC, and CI/CD
- [Phase 9 — Testing and Quality](09-testing-and-quality/README.md) — test strategy, integration testing, and static analysis
- [Phase 10 — Scaling](10-scaling/README.md) — replicas, sharding, rate limiting, and multi-region
- [Phase 11 — Advanced and Specialized](11-advanced/README.md) — runtime and database internals, and specialised workloads

## Capstones

- [Capstone projects](capstones/README.md) — notes and design decisions; the code lives in separate repositories

## Meta

- [superpowers/specs/](superpowers/specs/) — design docs
- [superpowers/plans/](superpowers/plans/) — implementation plans
