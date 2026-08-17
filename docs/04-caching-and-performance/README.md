# Phase 4 — Caching and Performance

The cache hierarchy, invalidation, and measuring before optimising.

Files are created when the topic is studied — see [CLAUDE.md](../../CLAUDE.md).

- [Cache theory: temporal/spatial locality, working set, hit ratio economics](cache-theory.md)
- [The full cache hierarchy: browser → CDN → reverse proxy → app → DB buffer pool](cache-hierarchy.md)
- [HTTP caching: `Cache-Control` directives, `ETag`, conditional requests, `Vary`](http-caching.md)
- [`stale-while-revalidate` / `stale-if-error`; CDN cache keys and purging](cdn-caching.md)
- [Redis fundamentals: data structures, TTLs, expiration, eviction policies (LRU/LFU/allkeys)](redis-fundamentals.md)
- [Redis beyond caching: sorted sets, HyperLogLog, streams, Lua scripting, pipelines](redis-beyond-caching.md)
- [Caching patterns: cache-aside, read-through, write-through, write-behind, refresh-ahead](caching-patterns.md)
- [Cache key design, namespacing, and versioned keys for bulk invalidation](cache-key-design.md)
- [Invalidation strategies: TTL, event-driven, write-invalidate — and why this is genuinely hard](cache-invalidation.md)
- [Cache stampede / thundering herd: locks, singleflight, probabilistic early expiry, jittered TTL](cache-stampede.md)
- [Negative caching, hot keys, cache penetration, sharding a cache](cache-failure-patterns.md)
- [Consistency implications: stale reads, read-your-own-writes, acceptable staleness budgets](cache-consistency.md)
- [In-process caching and its multi-instance pitfalls](in-process-caching.md)
- [Precomputation: materialized views, rollup tables, denormalized read models](precomputation.md)
- [Measuring first: profiling Python (`py-spy`, `cProfile`) and Node (`--prof`, clinic.js, flame graphs)](profiling.md)
- [Load testing with k6 / Locust: open vs closed models, realistic workloads](load-testing.md)
- [Latency statistics: why averages lie, p50/p95/p99, tail latency amplification](latency-statistics.md)
- [Little's Law, queueing theory intuition, utilization vs latency curves](queueing-theory.md)
- [Common wins: connection reuse, batching, N+1 elimination, payload size, compression](common-performance-wins.md)
- [Knowing when _not_ to cache](when-not-to-cache.md)
