# Phase 0 — Mental Models: What Actually Happens When a Request Arrives

The layer most CRUD experience skips entirely. Everything later depends on this.

Files are created when the topic is studied — see [CLAUDE.md](../../CLAUDE.md).

- [The lifecycle of a request: DNS → TCP → TLS → HTTP → your handler → response](request-lifecycle.md)
- [Process vs thread vs coroutine; what "concurrency" vs "parallelism" actually means](process-thread-coroutine.md)
- [CPU-bound vs I/O-bound work, and why the answer changes your entire architecture](cpu-bound-vs-io-bound.md)
- [Python's execution model: the GIL, `asyncio` event loop, when threads still help, `multiprocessing`](python-execution-model.md)
- [Node's execution model: libuv, event loop phases, the microtask queue, worker threads](node-execution-model.md)
- [WSGI vs ASGI (Gunicorn, Uvicorn, workers vs threads); Node clustering](wsgi-asgi-and-clustering.md)
- [Blocking the event loop: how to cause it, how to detect it, how to fix it in both runtimes](blocking-the-event-loop.md)
- [Sockets, file descriptors, and why "too many open files" happens](sockets-and-file-descriptors.md)
- [TCP fundamentals: handshake, keep-alive, connection reuse, head-of-line blocking](tcp-fundamentals.md)
- [TLS: handshake, certificates, chain of trust, termination points, mTLS preview](tls-fundamentals.md)
- [HTTP/1.1 vs HTTP/2 vs HTTP/3 — multiplexing, and what changes for your server](http-versions.md)
- [HTTP semantics in depth: safe vs idempotent methods, status code selection, header taxonomy](http-semantics.md)
- [Serialization formats: JSON, MessagePack, Protobuf, Avro — size, speed, schema evolution](serialization-formats.md)
- [Text encoding, Unicode, normalization, and the bugs they cause](text-encoding-and-unicode.md)
- [Time: UTC discipline, timezones, DST, monotonic vs wall clock, storing timestamps](time-and-timezones.md)
- [Identifiers: auto-increment vs UUIDv4 vs UUIDv7/ULID — index locality and enumeration risk](identifier-strategies.md)
- [Structured logging from day one (why `print`/`console.log` doesn't survive contact with prod)](structured-logging-basics.md)
