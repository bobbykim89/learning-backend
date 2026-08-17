# Phase 6 — Asynchronous and Distributed Systems

Queues, delivery semantics, and consistency.

Files are created when the topic is studied — see [CLAUDE.md](../../CLAUDE.md).

- [Why async processing exists: latency decoupling, load smoothing, failure isolation](why-async-processing.md)
- [Queues vs streams vs pub/sub — different primitives, different guarantees](queues-streams-pubsub.md)
- [Broker landscape: Redis Streams, RabbitMQ, SQS/SNS, Kafka, NATS — selection criteria](broker-landscape.md)
- [Task queues in practice: Celery, RQ, arq, Dramatiq; BullMQ, Graphile Worker](task-queue-libraries.md)
- [Durable execution engines: Temporal, Inngest, Restate — what problem they actually solve](durable-execution.md)
- [Delivery semantics: at-most-once, at-least-once, "exactly-once" and why it's a half-truth](delivery-semantics.md)
- [Idempotent consumers and deduplication strategies](idempotent-consumers.md)
- [Ordering guarantees, partitioning keys, consumer groups, rebalancing](ordering-and-partitioning.md)
- [Retries: exponential backoff with jitter, retry budgets, avoiding retry storms](retries-and-backoff.md)
- [Dead letter queues, poison messages, replay tooling](dead-letter-queues.md)
- [Scheduled and recurring work: cron correctness, missed runs, timezone traps](scheduled-jobs.md)
- [Distributed locks: Redlock, why it's contested, fencing tokens, lease-based approaches](distributed-locks.md)
- [Leader election and singleton workers](leader-election.md)
- [The dual-write problem and the transactional outbox pattern](transactional-outbox.md)
- [Change Data Capture (Debezium) as an integration primitive](change-data-capture.md)
- [Event-driven architecture: event notification vs event-carried state transfer](event-driven-architecture.md)
- [Event schema design, versioning, and the schema registry](event-schema-versioning.md)
- [CQRS: separating read and write models; when the complexity is justified](cqrs.md)
- [Event sourcing: append-only logs, projections, replay, snapshots — and the real costs](event-sourcing.md)
- [Sagas and compensating transactions; orchestration vs choreography](sagas-and-compensation.md)
- [Two-phase commit and why distributed transactions are usually avoided](two-phase-commit.md)
- [Consistency models: strong, eventual, causal, read-your-writes, monotonic reads](consistency-models.md)
- [CAP and PACELC — stated precisely, not as a slogan](cap-and-pacelc.md)
- [Time in distributed systems: clock skew, NTP, logical clocks, vector clocks](clocks-in-distributed-systems.md)
- [Consensus intuition: Raft, quorums, split brain](consensus-and-quorums.md)
- [Service communication: sync vs async, and the latency/coupling tradeoff](sync-vs-async-communication.md)
