---
title: Node's execution model
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

One thread running your JavaScript, a C library called libuv doing the waiting,
and a small thread pool for the operations that cannot be waited on
asynchronously by the OS.

This topic has no Python counterpart; the equivalent material is
[Python's execution model](python-execution-model.md). Node's version is stricter
— there is no threads-as-fallback option for ordinary code, because there is
exactly one JavaScript thread.

### The event loop, in phases

libuv runs a loop with ordered phases. Each phase has its own callback queue, and
the loop drains that queue before moving on:

1. **timers** — `setTimeout` and `setInterval` callbacks whose time has come
2. **pending callbacks** — deferred system callbacks, e.g. some TCP error cases
3. **idle / prepare** — internal
4. **poll** — the important one: wait for I/O events (epoll on Linux, kqueue on
   BSD/macOS) and run their callbacks. This is where the loop *blocks* when idle.
5. **check** — `setImmediate` callbacks
6. **close callbacks** — `'close'` events, e.g. a destroyed socket

Because `setImmediate` runs in **check** and `setTimeout(fn, 0)` runs in
**timers**, their relative order is a property of which phase the loop is in when
you schedule them — which is why the ordering of those two is famously
non-obvious and not something to depend on.

### Microtasks are not a phase

Promise callbacks do not live in any of those queues. There are two microtask
queues, drained **between** callbacks rather than as a phase:

- `process.nextTick` queue — drained first
- promise microtask queue (`.then`, `await` continuations, `queueMicrotask`) —
  drained after

Both are drained *to exhaustion* before the loop proceeds. A `nextTick` callback
that schedules another `nextTick` callback, forever, starves the loop completely
without ever blocking a single line of code.

The practical rule: **`await` yields to other tasks, but it does not yield to the
event loop's I/O phase in the way a blocking call would.** Microtasks jump the
queue ahead of timers and I/O.

### The thread pool, and what actually uses it

libuv keeps a small thread pool — **4 threads by default**, configurable with
`UV_THREADPOOL_SIZE`. It exists for operations with no good async OS interface:

- most `fs` operations
- `dns.lookup` (the `getaddrinfo` path), though `dns.resolve*` uses c-ares and
  does not
- some `crypto` (`pbkdf2`, `scrypt`, async `randomBytes`)
- `zlib`

**Network I/O does not use the pool.** Sockets are handled by epoll/kqueue in the
poll phase, which is why Node holds tens of thousands of connections on one
thread but four concurrent `fs.readFile` calls can queue behind each other.

That asymmetry surprises people: a service doing heavy file or hashing work has a
concurrency limit of 4 that nothing in the code suggests.

### worker_threads and cluster

- **`worker_threads`** — a separate V8 isolate with its own heap and its own event
  loop. Not shared memory by default: messages are structured-cloned, so passing
  large objects costs a copy. `SharedArrayBuffer` avoids the copy when you need
  genuine sharing. This is the answer for CPU-bound work.
- **`cluster`** — multiple *processes* sharing one listening socket, giving CPU
  parallelism plus crash isolation. Covered alongside the Python equivalents in
  [WSGI vs ASGI and clustering](wsgi-asgi-and-clustering.md).

## Why it exists (what came before)

The design is a direct answer to thread-per-connection servers. If most
connections are idle most of the time, a thread per connection spends memory on
stacks and scheduler time on context switches to accomplish waiting — and waiting
needs no thread at all if the OS can report readiness for many descriptors at
once, which is what `epoll` and `kqueue` do.

Node's contribution was not the event loop, which predates it by decades in C, but
making it the *default and only* concurrency model in a language whose callback
ergonomics made it tolerable — and later, with promises and `async`/`await`,
pleasant.

The cost of that choice is the single thread. Everything else in Node's
concurrency story — the thread pool, `worker_threads`, `cluster` — exists to work
around the consequence of having exactly one place to run JavaScript.

## Smallest example

Ordering first, because it exposes the queues:

```javascript
const fs = require("node:fs");

console.log("1 sync");

setTimeout(() => console.log("5 timer"), 0);        // timers phase
setImmediate(() => console.log("6 immediate"));     // check phase
fs.readFile(__filename, () => console.log("7 io")); // poll phase (thread pool)

Promise.resolve().then(() => console.log("4 promise microtask"));
process.nextTick(() => console.log("3 nextTick"));

console.log("2 sync end");
```

The two synchronous lines print first, then both microtask queues drain —
`nextTick` before promises — and only then does the loop advance to its phases.
Run it and confirm the order; the numbering above is the prediction, and the
timer/immediate pair in particular is worth verifying rather than trusting.

Then the failure, which is the part that matters operationally:

```javascript
const http = require("node:http");

function blockFor(ms) {                 // CPU-bound: no await, no yield
  const end = Date.now() + ms;
  while (Date.now() < end);             // holds the only JS thread
}

http.createServer((req, res) => {
  if (req.url === "/slow") blockFor(3000);
  res.end(req.url + "\n");
}).listen(8080);
```

Request `/slow` in one terminal and `/` in another. The second request does not
merely wait its turn politely — it is not *seen*, because the thread that would
call the handler is inside the `while` loop. Its connection sits in the kernel's
accept queue, exactly as described in
[the lifecycle of a request](request-lifecycle.md). Nothing in the `/` code path
is slow.

## Tradeoffs & when it's wrong

**The single-threaded loop** is right for I/O-bound services with many
connections, and its lack of shared-memory threads eliminates data races by
construction — a real correctness benefit that is easy to undervalue.

It is wrong wherever meaningful computation sits in the request path, and its
failure mode is unusually harsh: not a slow endpoint but a stalled process. There
is also no preemption, so a runaway loop cannot be interrupted by the runtime; a
health check will fail and the orchestrator will kill the process, which is the
only available remedy.

**`worker_threads`** is right for bounded CPU work — hashing, image processing,
parsing large payloads. It is wrong for work that needs constant access to large
shared state, because the structured-clone copy can exceed the computation.

**`cluster` / multiple processes** is right for using all cores and for crash
isolation, and is generally the first thing to reach for. It is wrong as a
substitute for fixing blocking code: N processes each blockable by one bad request
just raises the number of bad requests needed to stall the service.

What would change these: payload sizes, and whether the CPU work is bounded. A
100 ms CPU step at low traffic is invisible; the same step at 200 requests per
second is an outage.

## Failure modes & operational cost

- **Synchronous `fs` in a request path.** `readFileSync` blocks the loop for the
  whole read. Common in config or template loading that "only happens once" until
  it is in a hot path.
- **`JSON.parse` / `JSON.stringify` on large payloads.** Fully synchronous, and
  proportional to size. A 5 MB body is a multi-millisecond global stall.
- **Catastrophic regex backtracking.** Unbounded CPU inside a single call, driven
  by user input — a denial of service that looks like a hang.
- **`process.nextTick` recursion.** Starves the loop without blocking, so CPU
  looks busy and no I/O ever progresses.
- **Thread pool saturation.** More than `UV_THREADPOOL_SIZE` concurrent `fs`,
  `zlib`, or `pbkdf2` calls queue invisibly. Latency rises with no CPU
  saturation and no obvious culprit.
- **Unhandled promise rejections.** Terminate the process by default in current
  Node — a correctness improvement that turns a swallowed bug into a crash loop.
- **Error thrown in a callback with no handler.** Escapes to
  `uncaughtException`; the process state after that point is not trustworthy.
- **Memory: one heap for everything.** A leak in one request path degrades the
  whole process, and V8 heap limits apply per process, not per request.

## Open questions / to verify

Version-specific; not checked against primary docs in this session:

- The exact phase list and their names for the Node version in use, and whether
  the printed ordering of `setTimeout(0)` versus `setImmediate` in the script
  above is stable across runs.
- Whether `UV_THREADPOOL_SIZE` still defaults to 4 and what its current maximum
  is.
- Whether `dns.lookup` still routes through the thread pool while `dns.resolve*`
  uses c-ares, and whether any default has changed.
- The current default `cluster` scheduling policy on Linux (round-robin versus
  OS-decided) and how to change it.
- V8's default old-space heap limit on this machine, and whether Node still needs
  `--max-old-space-size` raised in containers.
- Whether `JSON.parse` has gained any incremental or streaming behaviour, or
  remains fully blocking for the whole input.

Candidate for `practiced`: run the blocking server above, measure `/` latency at
p50 and p99 while `/slow` is being hit, then move `blockFor` into a
`worker_thread` and measure again. Report the change in `/`'s p99, not `/slow`'s.

## Sources

- [Node — The event loop, timers, and `process.nextTick`](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- [Node — Don't block the event loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)
- [Node `worker_threads`](https://nodejs.org/api/worker_threads.html)
- [libuv design overview](https://docs.libuv.org/en/v1.x/design.html)
