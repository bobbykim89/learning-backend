---
title: WSGI vs ASGI, and clustering
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

The contract between a web server and your application code, and the process
model that runs many copies of it.

Two Python contracts, and Node's absence of one:

**WSGI** (PEP 3333) is synchronous. Your application is a callable taking an
`environ` dict and a `start_response` function, returning an iterable of bytes.
One call handles one request from start to finish. There is no way to express
"suspend here," which means **one worker handles one request at a time** — and no
way to express WebSockets, server-sent events, or anything long-lived.

**ASGI** is asynchronous. Your application is an async callable taking `scope`
(what this connection is), `receive` (await the next inbound message), and `send`
(emit an outbound message). Because it is message-based rather than
request-response, it covers HTTP, WebSockets, and a `lifespan` channel for
startup and shutdown — and because it is async, one worker holds many concurrent
requests.

**Node has no equivalent contract** because the runtime is already async and
already owns the HTTP server. `http.createServer(handler)` *is* the interface.
Frameworks like Express and Fastify are handlers, not adapters.

### The pre-fork process model

Both Python servers and Node's `cluster` use the same shape, and it is worth
holding precisely because it explains a class of production surprises:

1. A **master** process binds and listens on the port.
2. It **forks N workers**, which inherit the listening socket's file descriptor.
3. Every worker calls `accept()` on **the same** listening socket. The kernel
   decides which worker gets each connection.
4. The master supervises: restarts dead workers, enforces timeouts, orchestrates
   graceful reloads.

The consequence people trip over: **workers share nothing but that socket.** Each
has its own memory, its own caches, its own connection pools, its own counters.
Every in-process assumption you can make in development becomes false at N > 1.

## Why it exists (what came before)

Before WSGI, each Python web framework spoke to each web server through its own
adapter — mod_python, FastCGI shims, bespoke glue. PEP 333 (2003) standardised
one interface so any framework could run on any server. That decoupling is why
Django, Flask, and Pyramid could share a deployment ecosystem.

WSGI froze the synchronous request-response shape into the contract, so when
long-lived connections mattered — WebSockets, streaming, HTTP/2 — no amount of
server cleverness could express them. Workarounds existed (`gevent`
monkey-patching the standard library, so blocking calls became cooperative
without changing application code) but the contract itself could not represent a
suspended request.

ASGI was designed as the successor: same decoupling goal, message-based so it can
represent connections rather than only requests, async so a worker can hold
thousands.

Clustering predates all of it and needed no invention — `fork()` after `listen()`
is as old as Unix servers. Node adopted it because its single thread left no other
route to using more than one core.

## Smallest example

Both contracts with no framework, so the shape is visible.

```python
# WSGI: synchronous, one request at a time per worker.
#   run with: gunicorn app:wsgi_app
def wsgi_app(environ, start_response):
    body = b"hi from wsgi\n"
    start_response("200 OK", [
        ("Content-Type", "text/plain"),
        ("Content-Length", str(len(body))),
    ])
    return [body]


# ASGI: async, message-based, can represent connections.
#   run with: uvicorn app:asgi_app
async def asgi_app(scope, receive, send):
    if scope["type"] == "lifespan":            # startup/shutdown channel
        while True:
            message = await receive()
            if message["type"] == "lifespan.startup":
                await send({"type": "lifespan.startup.complete"})
            elif message["type"] == "lifespan.shutdown":
                await send({"type": "lifespan.shutdown.complete"})
                return

    if scope["type"] == "http":
        body = b"hi from asgi\n"
        await send({
            "type": "http.response.start",
            "status": 200,
            "headers": [
                (b"content-type", b"text/plain"),
                (b"content-length", str(len(body)).encode()),
            ],
        })
        await send({"type": "http.response.body", "body": body})
```

Note what ASGI's shape buys: the response is *two* messages, so a streaming
response is just more body messages. WSGI's `return [body]` cannot express
"more later" without the iterable staying alive and the worker staying occupied.

```javascript
// Node clustering: N processes, one shared listening socket.
import cluster from "node:cluster";
import http from "node:http";
import { availableParallelism } from "node:os";

if (cluster.isPrimary) {
  const n = availableParallelism();
  console.log(`primary ${process.pid}, forking ${n}`);
  for (let i = 0; i < n; i++) cluster.fork();
  cluster.on("exit", (worker) => {
    console.log(`worker ${worker.process.pid} died, replacing`);
    cluster.fork();                       // supervision, by hand
  });
} else {
  http.createServer((req, res) => {
    res.end(`served by ${process.pid}\n`);   // different pid per request
  }).listen(8080);
}
```

Hit it repeatedly with `curl` and watch the pid change. That is the kernel
distributing connections across workers that each called `accept()` on the same
descriptor.

## Sizing workers and threads

The heuristics circulate without their reasoning, so here is the reasoning.

**The real question is: how many requests must be in flight at once?** Little's
Law gives it directly — concurrency = throughput × latency. 200 requests/second
at 100 ms average latency means 20 requests in flight. That is the number your
worker and thread counts must cover, and it comes from measurement, not from core
count.

From there the workload's classification decides the shape (see
[CPU-bound vs I/O-bound](cpu-bound-vs-io-bound.md)):

- **CPU-bound**: workers ≈ cores. More workers than cores adds context switching
  and memory, not throughput.
- **I/O-bound, sync (WSGI)**: you need concurrency ≈ 20 from the example, and each
  worker gives 1. So either 20 workers, or fewer workers with threads
  (`gthread`), where each worker holds several waiting requests — the GIL is
  released during the wait, so this works.
- **I/O-bound, async (ASGI)**: one worker holds all 20 easily. Workers ≈ cores,
  for parallelism, not for concurrency.

Gunicorn's documented `(2 × cores) + 1` is a starting point for sync workers, not
a law, and it encodes an assumption about your latency and traffic that may not
hold. Measure, then set.

**The constraint that actually bites:** connection pools multiply by worker count.
Sixteen workers with a pool of 10 is 160 database connections, and Postgres
defaults to a max around 100. The service fails at startup or under load with
connection errors that look nothing like a sizing problem — see
[connection pooling](../02-data/performance/connection-pooling.md).

## Tradeoffs & when it's wrong

**WSGI** is right for ordinary request-response services, and its simplicity is a
real asset: no way to block the loop, because there is no shared loop; profiling
and debugging are straightforward; any library works. It is wrong when you need
long-lived connections, or when connection counts are high enough that a worker
per concurrent request is too expensive.

**ASGI** is right for high connection counts, streaming, WebSockets, and
fan-out request patterns. It is wrong when your dependencies are synchronous, and
its failure mode is worse than WSGI's: one blocking call degrades every
connection on that worker rather than one request.

Running a sync framework under an async server does not make it async, and
running async code under `gthread` workers gets you the hazards of both.

**Clustering** is right nearly always — it uses the cores you paid for and
survives a worker crash. It is wrong as a fix for blocking code, and it silently
breaks any design that assumed one process: in-memory caches, in-memory rate
limiters, in-memory session stores, module-level counters, scheduled jobs that
now run N times.

What would change these choices: connection count, whether any dependency lacks
an async driver, and whether the request path holds CPU work.

## Failure modes & operational cost

- **State assumed shared, actually per-worker.** Caches disagree, rate limits are
  N times looser than configured, a cron-like scheduler fires N times. Correct in
  development, wrong in production, and often silent.
- **Connection pool multiplication.** Workers × pool size exceeds the database's
  limit.
- **Resources created before `fork()`.** A database connection or socket opened in
  the parent and inherited by children is *the same socket* in several processes,
  producing interleaved protocol traffic and corruption. Create connections after
  fork, or use the server's post-fork hook. This is the price of `preload_app`'s
  memory savings.
- **Worker timeout versus slow requests.** A sync worker exceeding the master's
  timeout is killed mid-request. Raising the timeout to accommodate a slow
  endpoint also delays detection of genuinely hung workers.
- **Blocking in an async worker.** No timeout fires, because the worker is
  responsive to the master's heartbeat; it is only your requests that are stalled.
- **Deep health checks.** A health endpoint that touches the database means one
  slow database marks every worker unhealthy and the orchestrator restarts a
  fleet that was fine.
- **Memory per worker.** N workers is roughly N times the resident set, minus
  whatever copy-on-write keeps shared. Container limits are enforced per pod, not
  per worker, so a worker count that fits on a laptop OOMs in production.
- **Graceful shutdown ignored.** Workers killed without draining drop in-flight
  requests on every deploy.

## Open questions / to verify

Version- and deployment-specific; not verified against primary docs here:

- Gunicorn's current default worker class, default worker count, and default
  timeout, and whether the timeout is measured differently for async workers.
- Whether running `uvicorn --workers N` uses its own supervisor rather than
  Gunicorn's, and which is currently recommended for production.
- Node's default `cluster` scheduling policy on Linux, and whether
  `SO_REUSEPORT`-based distribution is available or preferable to the primary
  distributing handles.
- Whether `availableParallelism()` respects container CPU limits, or reports host
  cores — this determines whether the example above over-forks in a container.
- Postgres's `max_connections` default on the version in use, to make the
  multiplication arithmetic concrete.
- Whether ASGI's `lifespan` protocol is optional in current Uvicorn, and what
  happens to an app that does not implement it.

Candidate for `practiced`: deploy the same I/O-bound app twice — WSGI with sync
workers, ASGI with async workers — put the same load through both, and find the
concurrency level at which the WSGI version's latency departs from the ASGI
version's. Then add one blocking call to the ASGI version and watch it lose.

## Sources

- [PEP 3333 — WSGI](https://peps.python.org/pep-3333/)
- [ASGI specification](https://asgi.readthedocs.io/en/latest/specs/main.html)
- [Gunicorn — design and worker types](https://docs.gunicorn.org/en/stable/design.html)
- [Uvicorn deployment](https://www.uvicorn.org/deployment/)
- [Node `cluster`](https://nodejs.org/api/cluster.html)
