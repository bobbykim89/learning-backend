# Phase 7 — Reliability and Operations

Failure handling, observability, and SLOs.

Files are created when the topic is studied — see [CLAUDE.md](../../CLAUDE.md).

- [Failure taxonomy: crash, partial, gray failures, cascading failure](failure-taxonomy.md)
- [Timeouts everywhere; deadline propagation and request budgets](timeouts-and-deadlines.md)
- [Circuit breakers, bulkheads, load shedding, admission control](resilience-patterns.md)
- [Backpressure: propagating it correctly instead of buffering to death](backpressure.md)
- [Graceful shutdown: SIGTERM handling, connection draining, in-flight work](graceful-shutdown.md)
- [Health checks: liveness vs readiness vs startup; the danger of deep health checks](health-checks.md)
- [Degraded modes and fallbacks; designing what "partially working" looks like](degraded-modes.md)
- [Observability pillars: logs, metrics, traces, and how they compose](observability-pillars.md)
- [Structured logging, correlation/request IDs, log levels, sampling, cost control](logging-in-production.md)
- [Metrics: counters/gauges/histograms, cardinality explosions, RED and USE methods](metrics.md)
- [Distributed tracing and OpenTelemetry: spans, context propagation, instrumenting Python and Node](distributed-tracing.md)
- [Dashboards that answer questions; alerting on symptoms, not causes](dashboards-and-alerting.md)
- [SLIs, SLOs, error budgets — and using them to make engineering decisions](slos-and-error-budgets.md)
- [On-call practice: runbooks, escalation, incident command, blameless postmortems](on-call-and-incidents.md)
- [Backups: full vs incremental, point-in-time recovery, and testing restores](backups.md)
- [Disaster recovery: RPO/RTO, failover strategy, multi-AZ vs multi-region](disaster-recovery.md)
- [Chaos engineering and game days](chaos-engineering.md)
- [Capacity planning and forecasting](capacity-planning.md)
