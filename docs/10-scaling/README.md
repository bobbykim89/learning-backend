# Phase 10 — Scaling

Replicas, sharding, rate limiting, and multi-region.

Files are created when the topic is studied — see [CLAUDE.md](../../CLAUDE.md).

- [Vertical vs horizontal scaling; knowing which problem you have](vertical-vs-horizontal-scaling.md)
- [Statelessness as an enabler; where state actually lives](statelessness.md)
- [Session storage and sticky sessions — and why to avoid them](session-storage.md)
- [Read replicas: replication lag, read-your-writes, routing reads safely](read-replicas.md)
- [Failover, promotion, synchronous vs asynchronous replication](replication-and-failover.md)
- [Partitioning within a database (Postgres declarative partitioning)](table-partitioning.md)
- [Sharding: key selection, hash vs range, consistent hashing, resharding pain](sharding.md)
- [Cross-shard queries, distributed joins, global uniqueness under sharding](cross-shard-queries.md)
- [Distributed SQL (CockroachDB, Vitess, Citus, Aurora) — what they buy you](distributed-sql.md)
- [Rate limiting algorithms: fixed/sliding window, token bucket, leaky bucket](rate-limiting-algorithms.md)
- [Distributed rate limiting and quota enforcement without a bottleneck](distributed-rate-limiting.md)
- [Fan-out strategies: write-time vs read-time (the timeline problem)](fan-out-strategies.md)
- [Multi-region: latency, data residency, active-active vs active-passive, conflict resolution](multi-region.md)
- [Edge compute and CDN-level logic](edge-compute.md)
- [Noisy neighbors, per-tenant isolation, fairness and prioritization](tenant-isolation-and-fairness.md)
- [The monolith → services decision: coupling, team topology, transaction boundaries](monolith-to-services.md)
- [Strangler fig migrations and extracting a service safely](strangler-fig-migration.md)

> `rate-limiting-algorithms.md` and `distributed-rate-limiting.md` cover mechanism and
> coordination; rate limiting as an abuse-prevention control lives in
> [Phase 3 — Rate limiting and abuse prevention](../03-security/data-protection-and-operations/rate-limiting-as-security.md).
