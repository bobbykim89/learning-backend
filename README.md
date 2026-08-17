# Backend Engineering: Beginner → Advanced

A study roadmap for a frontend-leaning senior engineer moving into depth on backend systems.
Primary languages: **Python** and **TypeScript/Node**.

**How to use this:** each bullet is scoped to be one focused study session (30–90 min). Work top-down within a phase, but phases 3–5 can be interleaved once Phase 2 is solid. Prompt patterns for AI-assisted study are at the bottom.

---

## Phase 0 — Mental Models: What Actually Happens When a Request Arrives

The layer most CRUD experience skips entirely. Everything later depends on this.

- The lifecycle of a request: DNS → TCP → TLS → HTTP → your handler → response
- Process vs thread vs coroutine; what "concurrency" vs "parallelism" actually means
- CPU-bound vs I/O-bound work, and why the answer changes your entire architecture
- Python's execution model: the GIL, `asyncio` event loop, when threads still help, `multiprocessing`
- Node's execution model: libuv, event loop phases, the microtask queue, worker threads
- WSGI vs ASGI (Gunicorn, Uvicorn, workers vs threads); Node clustering
- Blocking the event loop: how to cause it, how to detect it, how to fix it in both runtimes
- Sockets, file descriptors, and why "too many open files" happens
- TCP fundamentals: handshake, keep-alive, connection reuse, head-of-line blocking
- TLS: handshake, certificates, chain of trust, termination points, mTLS preview
- HTTP/1.1 vs HTTP/2 vs HTTP/3 — multiplexing, and what changes for your server
- HTTP semantics in depth: safe vs idempotent methods, status code selection, header taxonomy
- Serialization formats: JSON, MessagePack, Protobuf, Avro — size, speed, schema evolution
- Text encoding, Unicode, normalization, and the bugs they cause
- Time: UTC discipline, timezones, DST, monotonic vs wall clock, storing timestamps
- Identifiers: auto-increment vs UUIDv4 vs UUIDv7/ULID — index locality and enumeration risk
- Structured logging from day one (why `print`/`console.log` doesn't survive contact with prod)

---

## Phase 1 — Application Architecture and Code Design

Where dependency injection, testability, and "how do I structure this" live.

- Layered architecture: transport → service/use-case → repository → data source
- Why the layers exist: dependency direction, and what breaks when you skip them
- Inversion of Control and Dependency Injection — the concept, independent of frameworks
- Manual/constructor DI in plain Python and plain TypeScript (start here, no library)
- DI in practice: FastAPI `Depends`, `dependency-injector`; NestJS, tsyringe, InversifyJS
- Service lifetimes: singleton vs scoped vs transient, and request-scoped state
- DI tradeoffs: when a container is overkill, magic vs explicitness, startup cost
- Ports and adapters (hexagonal architecture); when it pays off and when it's ceremony
- SOLID, pragmatically — which principles matter daily, which are academic
- Composition over inheritance; protocols/structural typing (Python `Protocol`, TS interfaces)
- Domain modeling: entities, value objects, aggregates, invariants
- DTOs vs domain models vs DB rows — why collapsing them hurts later
- Validation at the boundary: Pydantic v2, Zod — parse, don't validate
- Error design: exception hierarchies vs result types, expected vs exceptional failures
- Error taxonomy for APIs: client vs server, retryable vs terminal, RFC 9457 problem details
- Configuration and the 12-factor app; typed config, fail-fast on startup
- Feature flags as an architectural tool
- Project structure: layer-first vs feature-first, the modular monolith
- When to split a service — and the strong case for _not_ doing microservices yet
- Middleware/interceptor pipelines: cross-cutting concerns done right
- Writing an Architecture Decision Record (ADR) — capturing tradeoffs, not just choices

---

## Phase 2 — Data: Relational Depth First

The single highest-leverage area. Most backend performance problems are database problems.

### Relational fundamentals

- Relational modeling: normalization to 3NF, and deliberate denormalization
- Keys and constraints: primary, foreign, unique, check, NOT NULL — pushing invariants to the DB
- Data types that matter: numeric vs float for money, `timestamptz`, JSONB, arrays, enums
- SQL beyond CRUD: joins, aggregates, `GROUP BY`, window functions, CTEs, `LATERAL`
- Upserts, `RETURNING`, bulk operations, `COPY`/batch insert

### Performance

- How B-tree indexes work; why an index is a sorted copy with a cost
- Composite index column order, partial indexes, covering indexes, index-only scans
- Other index types: GIN, GiST, BRIN, hash — and when each applies
- Reading `EXPLAIN ANALYZE`: seq scan vs index scan, nested loop vs hash vs merge join
- Query planner statistics, cardinality estimation, when the planner gets it wrong
- Slow query logs, `pg_stat_statements`, finding the real bottleneck
- Connection pooling: pool sizing math, PgBouncer, pool exhaustion symptoms

### Transactions and concurrency

- ACID, concretely — what each letter buys you
- Isolation levels and their anomalies: dirty read, non-repeatable read, phantom, write skew
- MVCC in Postgres; bloat and `VACUUM`
- Locking: row vs table, `SELECT FOR UPDATE`, lock ordering, deadlock detection
- Optimistic vs pessimistic concurrency control; version columns
- Transaction scope: where to begin/commit relative to your service layer
- Long transactions and why they're a production hazard

### ORMs and access layers

- What an ORM actually does: identity map, unit of work, change tracking, lazy loading
- The N+1 problem — detecting it, and eager loading strategies
- SQLAlchemy 2.0 (Core vs ORM), Prisma vs Drizzle vs Kysely vs TypeORM — tradeoffs
- The repository pattern: value, cost, and when it becomes an anti-pattern
- Dropping to raw SQL safely: parameterization, typed query builders
- Query result mapping and avoiding over-fetching

### Schema evolution

- Migration tooling: Alembic, Prisma Migrate, Drizzle Kit
- Zero-downtime schema changes: expand → migrate → contract
- Dangerous migrations: locking DDL, adding NOT NULL, changing types, big backfills
- Backfill strategies: batched, resumable, throttled
- Running migrations in CI/CD: ordering vs deploys, rollback strategy

### Modeling patterns

- Soft deletes: the case for and against
- Audit trails, history tables, temporal/bitemporal data
- Multi-tenancy: shared schema vs schema-per-tenant vs DB-per-tenant, and row-level security
- Idempotency keys and exactly-once request handling
- Hierarchies, graphs, and recursive queries in SQL

### Beyond relational

- When a relational DB is the wrong tool (and how rarely that's true)
- Document stores (MongoDB/DynamoDB): access-pattern-first modeling, single-table design
- Key-value, wide-column, time-series, graph databases — the fit for each
- Full-text search: Postgres `tsvector` vs Elasticsearch/OpenSearch tradeoffs
- Vector search and embeddings storage (pgvector and alternatives)
- Object storage: S3 semantics, consistency, lifecycle, storage classes
- Polyglot persistence and the cost of every additional datastore you adopt

---

## Phase 3 — Security

Not a checklist to bolt on. A design constraint that shapes every layer.

### Foundations

- Threat modeling with STRIDE; trust boundaries; thinking like an attacker
- Defense in depth, least privilege, fail-closed, secure defaults
- Authentication vs authorization vs accounting — precise definitions

### Authentication

- Password storage: Argon2id/bcrypt, salting, peppering, work factors, rehashing on login
- Session cookies vs JWTs: the real tradeoffs, and why "stateless" isn't free
- Cookie security: `HttpOnly`, `Secure`, `SameSite`, domain/path scoping
- JWT correctly: signing algorithms, `alg` confusion, claims validation, key rotation via JWKS
- Refresh tokens: rotation, reuse detection, revocation, token families
- OAuth 2.1 and OIDC: authorization code + PKCE, client credentials, device flow
- Identity providers: what you delegate to Auth0/Clerk/Cognito/Keycloak and what stays yours
- MFA: TOTP, WebAuthn/passkeys, recovery codes
- Account lifecycle security: verification, password reset, enumeration resistance, session invalidation

### Authorization

- RBAC, ABAC, ReBAC (Google Zanzibar model) — expressiveness vs complexity
- Enforcement points: middleware vs service layer vs database (RLS)
- IDOR / broken object-level authorization — the most common real-world API vuln
- Policy engines: OPA/Rego, Cedar, OpenFGA
- Service-to-service auth: mTLS, signed JWTs, workload identity

### Application vulnerabilities

- OWASP API Security Top 10, worked through with concrete examples
- Injection: SQL, NoSQL, command, template, LDAP — parameterization as the fix
- SSRF: metadata endpoint attacks, allowlists, DNS rebinding
- CSRF: when it applies to APIs, double-submit vs SameSite
- XSS from the backend's perspective: content types, sanitization boundaries, `nosniff`
- Insecure deserialization: pickle, YAML, prototype pollution
- Mass assignment / over-posting; explicit allowlists
- Path traversal, ZIP slip, unrestricted file upload
- Timing attacks and constant-time comparison
- CORS actually understood: preflight, credentials, why `*` fails, common misconfigurations

### Data protection and operations

- Secrets management: Vault, cloud secret managers; never in env files in git
- Encryption at rest vs in transit vs application-level; envelope encryption, KMS, key rotation
- PII handling, data classification, tokenization, log redaction
- Rate limiting and abuse prevention as security controls
- Supply chain: lockfiles, integrity hashes, SBOM, `npm audit`/`pip-audit`, Dependabot, typosquatting
- Container and runtime hardening: non-root, read-only FS, minimal images
- Security headers and HTTPS enforcement (HSTS, CSP as a backend concern)
- Audit logging: what to record, tamper-resistance, retention
- Compliance orientation: GDPR/CCPA basics, SOC 2 concepts, data residency, right to erasure
- Responsible disclosure, CVE monitoring, patch cadence

---

## Phase 4 — Caching and Performance

- Cache theory: temporal/spatial locality, working set, hit ratio economics
- The full cache hierarchy: browser → CDN → reverse proxy → app → DB buffer pool
- HTTP caching: `Cache-Control` directives, `ETag`, conditional requests, `Vary`
- `stale-while-revalidate` / `stale-if-error`; CDN cache keys and purging
- Redis fundamentals: data structures, TTLs, expiration, eviction policies (LRU/LFU/allkeys)
- Redis beyond caching: sorted sets, HyperLogLog, streams, Lua scripting, pipelines
- Caching patterns: cache-aside, read-through, write-through, write-behind, refresh-ahead
- Cache key design, namespacing, and versioned keys for bulk invalidation
- Invalidation strategies: TTL, event-driven, write-invalidate — and why this is genuinely hard
- Cache stampede / thundering herd: locks, singleflight, probabilistic early expiry, jittered TTL
- Negative caching, hot keys, cache penetration, sharding a cache
- Consistency implications: stale reads, read-your-own-writes, acceptable staleness budgets
- In-process caching and its multi-instance pitfalls
- Precomputation: materialized views, rollup tables, denormalized read models
- Measuring first: profiling Python (`py-spy`, `cProfile`) and Node (`--prof`, clinic.js, flame graphs)
- Load testing with k6 / Locust: open vs closed models, realistic workloads
- Latency statistics: why averages lie, p50/p95/p99, tail latency amplification
- Little's Law, queueing theory intuition, utilization vs latency curves
- Common wins: connection reuse, batching, N+1 elimination, payload size, compression
- Knowing when _not_ to cache

---

## Phase 5 — API Design and Contracts

- Resource modeling, URL design, and Richardson maturity levels
- Pagination: offset vs keyset/cursor, stable ordering, total counts
- Filtering, sorting, sparse fieldsets, expansion — without reinventing GraphQL badly
- Versioning strategies: URL, header, media type; deprecation policy and sunset headers
- Idempotency for unsafe methods; idempotency keys end-to-end
- Bulk and batch endpoints; partial success semantics
- Long-running operations: 202 + status resource, polling vs callbacks
- OpenAPI-first design, codegen for clients and servers, contract testing (Pact)
- Framework comparison: FastAPI, Litestar, Django REST vs Express, Fastify, NestJS, Hono
- GraphQL: schema design, resolvers, N+1 and DataLoader, pagination conventions
- GraphQL in production: depth/complexity limits, persisted queries, caching difficulties, federation
- gRPC and Protobuf: streaming modes, schema evolution, when RPC beats REST
- tRPC and type-safe RPC in a TypeScript monorepo
- Realtime: polling vs long polling vs SSE vs WebSockets — tradeoffs and scaling each
- WebSocket production concerns: auth, heartbeats, reconnection, fan-out via pub/sub, backpressure
- Webhooks you _send_: signing (HMAC), retries, ordering, replay protection, subscriber management
- Webhooks you _receive_: verification, idempotency, fast-ack + async processing
- File handling: presigned URLs, multipart and resumable uploads, streaming, content validation
- API gateways: routing, auth offload, rate limiting, request transformation
- BFF pattern — where your frontend experience gives you an edge

---

## Phase 6 — Asynchronous and Distributed Systems

- Why async processing exists: latency decoupling, load smoothing, failure isolation
- Queues vs streams vs pub/sub — different primitives, different guarantees
- Broker landscape: Redis Streams, RabbitMQ, SQS/SNS, Kafka, NATS — selection criteria
- Task queues in practice: Celery, RQ, arq, Dramatiq; BullMQ, Graphile Worker
- Durable execution engines: Temporal, Inngest, Restate — what problem they actually solve
- Delivery semantics: at-most-once, at-least-once, "exactly-once" and why it's a half-truth
- Idempotent consumers and deduplication strategies
- Ordering guarantees, partitioning keys, consumer groups, rebalancing
- Retries: exponential backoff with jitter, retry budgets, avoiding retry storms
- Dead letter queues, poison messages, replay tooling
- Scheduled and recurring work: cron correctness, missed runs, timezone traps
- Distributed locks: Redlock, why it's contested, fencing tokens, lease-based approaches
- Leader election and singleton workers
- The dual-write problem and the transactional outbox pattern
- Change Data Capture (Debezium) as an integration primitive
- Event-driven architecture: event notification vs event-carried state transfer
- Event schema design, versioning, and the schema registry
- CQRS: separating read and write models; when the complexity is justified
- Event sourcing: append-only logs, projections, replay, snapshots — and the real costs
- Sagas and compensating transactions; orchestration vs choreography
- Two-phase commit and why distributed transactions are usually avoided
- Consistency models: strong, eventual, causal, read-your-writes, monotonic reads
- CAP and PACELC — stated precisely, not as a slogan
- Time in distributed systems: clock skew, NTP, logical clocks, vector clocks
- Consensus intuition: Raft, quorums, split brain
- Service communication: sync vs async, and the latency/coupling tradeoff

---

## Phase 7 — Reliability and Operations

- Failure taxonomy: crash, partial, gray failures, cascading failure
- Timeouts everywhere; deadline propagation and request budgets
- Circuit breakers, bulkheads, load shedding, admission control
- Backpressure: propagating it correctly instead of buffering to death
- Graceful shutdown: SIGTERM handling, connection draining, in-flight work
- Health checks: liveness vs readiness vs startup; the danger of deep health checks
- Degraded modes and fallbacks; designing what "partially working" looks like
- Observability pillars: logs, metrics, traces, and how they compose
- Structured logging, correlation/request IDs, log levels, sampling, cost control
- Metrics: counters/gauges/histograms, cardinality explosions, RED and USE methods
- Distributed tracing and OpenTelemetry: spans, context propagation, instrumenting Python and Node
- Dashboards that answer questions; alerting on symptoms, not causes
- SLIs, SLOs, error budgets — and using them to make engineering decisions
- On-call practice: runbooks, escalation, incident command, blameless postmortems
- Backups: full vs incremental, point-in-time recovery, and testing restores
- Disaster recovery: RPO/RTO, failover strategy, multi-AZ vs multi-region
- Chaos engineering and game days
- Capacity planning and forecasting

---

## Phase 8 — Infrastructure and Delivery

- Containers: images, layers, caching, multi-stage builds for Python and Node
- Image hygiene: slim/distroless bases, non-root users, `.dockerignore`, reproducible builds
- Docker Compose for local development that mirrors production
- Orchestration concepts: pods, deployments, services, ingress, configmaps, secrets
- Resource requests/limits, autoscaling (HPA), OOM kills, CPU throttling
- Simpler alternatives: ECS/Fargate, Fly.io, Render, Railway — and when they're the right call
- Serverless: cold starts, execution limits, connection pooling problems, cost model
- Load balancers L4 vs L7, reverse proxies (Nginx, Caddy, Envoy), service mesh overview
- Networking: VPCs, subnets, security groups, private links, egress control
- DNS operationally: TTLs, failover, blue/green cutover
- Infrastructure as Code: Terraform or Pulumi, state management, drift
- CI/CD pipelines: build once, promote artifacts, environment parity
- Deployment strategies: rolling, blue/green, canary, progressive delivery with flags
- Ephemeral preview environments and seeded test data
- Secrets in CI, OIDC-based cloud auth, least-privilege pipelines
- Cost engineering: egress charges, storage tiers, right-sizing, the cost of idle capacity

---

## Phase 9 — Testing and Quality

- Test strategy: pyramid vs trophy; what each level is actually for
- Test doubles: stub, mock, fake, spy — and how DI makes them unnecessary as often as possible
- Unit testing business logic with no I/O; designing for that
- Integration testing against real dependencies with Testcontainers
- Database test strategy: transactional rollback, truncation, migrations in tests, factories
- Deterministic tests: freezing time, seeded randomness, controlling concurrency
- Testing async code and background workers
- Contract testing between services; consumer-driven contracts
- Property-based testing: Hypothesis (Python), fast-check (TS)
- Snapshot and approval testing — the good and the trap
- Mutation testing to measure suite quality
- Load, soak, spike, and stress tests as part of the pipeline
- Static analysis: mypy/pyright strict, TypeScript strict mode, ruff, ESLint
- Making illegal states unrepresentable with the type system
- Test flakiness: causes, quarantine, and fixing rather than retrying

---

## Phase 10 — Scaling

- Vertical vs horizontal scaling; knowing which problem you have
- Statelessness as an enabler; where state actually lives
- Session storage and sticky sessions — and why to avoid them
- Read replicas: replication lag, read-your-writes, routing reads safely
- Failover, promotion, synchronous vs asynchronous replication
- Partitioning within a database (Postgres declarative partitioning)
- Sharding: key selection, hash vs range, consistent hashing, resharding pain
- Cross-shard queries, distributed joins, global uniqueness under sharding
- Distributed SQL (CockroachDB, Vitess, Citus, Aurora) — what they buy you
- Rate limiting algorithms: fixed/sliding window, token bucket, leaky bucket
- Distributed rate limiting and quota enforcement without a bottleneck
- Fan-out strategies: write-time vs read-time (the timeline problem)
- Multi-region: latency, data residency, active-active vs active-passive, conflict resolution
- Edge compute and CDN-level logic
- Noisy neighbors, per-tenant isolation, fairness and prioritization
- The monolith → services decision: coupling, team topology, transaction boundaries
- Strangler fig migrations and extracting a service safely

---

## Phase 11 — Advanced and Specialized

- Python performance: `asyncio` internals, uvloop, structured concurrency, TaskGroups
- Python at the edges: C extensions, Cython, Rust via PyO3, free-threaded builds
- Python memory: reference counting, GC, leak hunting, `tracemalloc`
- Node internals: event loop phases in detail, streams and backpressure, worker threads
- Node memory: V8 heap, GC pauses, heap snapshots, leak patterns
- Database internals: storage layout, WAL, buffer pool, B-tree vs LSM tree, compaction
- Writing middleware, an ASGI app, or a small HTTP server from scratch
- Designing a protocol: framing, versioning, backwards/forwards compatibility
- OLTP vs OLAP; data warehouses, ELT, dbt, analytics without wrecking production
- Streaming data processing and materialized views over event streams
- Serving ML/LLM workloads: batching, GPU vs CPU constraints, queues, timeouts, streaming responses
- Retrieval infrastructure: vector stores, hybrid search, embedding pipelines
- Platform engineering: golden paths, internal libraries, developer experience
- Buy vs build, and evaluating vendors and open-source dependencies
- Writing RFCs and design docs; running a design review
- System design practice: sketching, estimating, and defending tradeoffs out loud

---

## Capstone Projects

Build these in order. Each one forces the concepts from its phases into your hands.

1. **URL shortener with real constraints** — ID generation, caching, rate limiting, analytics via async writes. _(Phases 0, 2, 4)_
2. **Multi-tenant task API** — layered architecture with DI, RBAC, RLS, cursor pagination, OpenAPI, full test suite. _(Phases 1, 3, 5, 9)_
3. **Payments-style ledger** — double-entry accounting, transactional integrity, idempotency keys, exactly-once webhooks, reconciliation. _(Phases 2, 5, 6)_
4. **Event-driven order pipeline** — outbox pattern, message broker, saga with compensation, DLQ, replay tooling. _(Phase 6)_
5. **High-throughput read service** — multi-layer caching, stampede protection, load tested to a documented p99 target with tracing and dashboards. _(Phases 4, 7)_
6. **Ship one of the above properly** — containerized, IaC-provisioned, CI/CD with zero-downtime migrations, SLOs, alerts, and a tested restore. _(Phases 7, 8)_

---

## Using This With an AI Tutor

The failure mode of AI-assisted learning is receiving fluent summaries and mistaking recognition for understanding. Structure sessions to prevent that:

**Per topic, ask in this order:**

1. _"Explain X, then show me the smallest code example in Python and in TypeScript."_
2. _"What problem did X exist to solve? What did people do before it?"_
3. _"What are the tradeoffs of X — what does it cost me, and when is it the wrong choice?"_
4. _"Show me a realistic implementation with a subtle bug in it, and let me find it."_
5. _"Quiz me on X with scenario questions, not definitions. Don't give me the answer until I've committed to one."_

**Specifically for tradeoffs** (your stated gap): pick a concrete decision and ask for a defense of _both_ sides before any recommendation — e.g. "Argue for JWTs, then argue for server-side sessions, then tell me which questions about my system determine the answer."

**Guardrails:** ask for the failure modes and the operational cost of anything before adopting it. Ask "what would change your answer?" Verify anything version-specific against primary docs — library APIs and defaults drift, and confident-sounding detail is where models are least reliable.

**Track it:** set `status: practiced` in the topic doc's frontmatter, but only after you've built something with the concept, not after reading about it.
