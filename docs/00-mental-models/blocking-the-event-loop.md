---
title: Blocking the event loop
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

Any stretch of synchronous work that runs between two yield points, long enough
that the event loop cannot service anything else while it runs.

The precise definition matters because the loop has no way to interrupt you.
Cooperative scheduling means control returns to the loop only when your code
yields — `await` in Python and JavaScript. Between two yields, your code owns the
thread absolutely. That is the property that makes coroutines cheap, and it is
the same property that makes one careless line a service-wide outage.

The signature is what makes this worth its own topic: **the latency lands
somewhere other than the code that caused it.** A 200 ms synchronous stretch in
one rarely-used endpoint adds up to 200 ms to every other request in flight on
that worker. Your dashboards show a global p99 rise. The endpoint responsible
looks fine, because it was busy, not waiting.

### How to cause it

Not an exhaustive list, but these are the ones that actually happen:

- **CPU work in the request path** — see
  [CPU-bound vs I/O-bound](cpu-bound-vs-io-bound.md). Serialising a large
  response, resizing an image, hashing a password.
- **A synchronous driver call.** `requests` inside `async def`, a DB library with
  no async support, `time.sleep`, `readFileSync`.
- **`JSON.parse` / `json.loads` on a large payload.** Fully synchronous and
  proportional to size.
- **Regex with catastrophic backtracking.** Unbounded CPU, driven by user input.
- **Synchronous crypto.** `crypto.pbkdf2Sync`, `bcrypt` without a thread.
- **Compression** of a large body inline.
- **A large sort or aggregation** in application code over a big result set.
- **Logging that fsyncs**, or writes to a slow destination synchronously.
- **`process.nextTick` recursion** in Node — starves the loop without blocking a
  single line, because the microtask queue drains to exhaustion before the loop
  advances.

### How to detect it

The critical move is measuring **loop delay** — how long the loop waited to get
control back — rather than endpoint latency, because endpoint latency attributes
the cost to the victim rather than the culprit.

**Node.** `perf_hooks.monitorEventLoopDelay()` returns a histogram you can expose
as a metric. A crude but effective version, useful when you have nothing:

```javascript
import { monitorEventLoopDelay } from "node:perf_hooks";

const h = monitorEventLoopDelay({ resolution: 10 });
h.enable();
setInterval(() => {
  console.log(`loop delay p99=${(h.percentile(99) / 1e6).toFixed(1)}ms max=${(h.max / 1e6).toFixed(1)}ms`);
  h.reset();
}, 5000);
```

**Python.** `asyncio` debug mode logs any callback that runs longer than
`loop.slow_callback_duration`, which is the same idea from the other direction:

```python
import asyncio, logging

logging.basicConfig(level=logging.WARNING)

async def main():
    loop = asyncio.get_running_loop()
    loop.set_debug(True)
    loop.slow_callback_duration = 0.05     # warn on anything over 50ms
    ...

asyncio.run(main())
```

Also available: `py-spy dump --pid N` to see what a stalled process is executing
right now, without restarting it or adding instrumentation — the single most
useful tool when a production process is wedged.

**The symptom pattern**, worth recognising before you have metrics: global
latency rise across unrelated endpoints, health checks flapping under load,
timeouts on calls that are not slow, and CPU that looks busy while throughput
falls.

## Why it exists (what came before)

This failure mode is the bill for the C10k solution. Thread-per-connection
servers could not be blocked service-wide by one slow handler, because the
kernel preempts threads — a runaway handler stole CPU share but could not stop
other threads from running.

Moving scheduling into user space bought cheap waiting and gave up preemption.
There is no timer interrupt that will take control away from your coroutine, so
the runtime cannot protect you. The discipline that used to be the kernel's job
is now yours.

## Smallest example

The fault, then two fixes, then a fix that isn't one.

```python
import asyncio, time

def hash_password(pw: str) -> str:      # stand-in for real CPU work
    t = 0
    for i in range(5_000_000):
        t += i
    return str(t)

# WRONG: async def does not make a synchronous body asynchronous
async def register_bad(pw):
    return hash_password(pw)

# RIGHT for blocking/CPU calls: push it off the loop
async def register_good(pw):
    return await asyncio.to_thread(hash_password, pw)

async def heartbeat():
    """Prints every 100ms. Gaps in the output ARE the blocking."""
    while True:
        print(f"tick {time.strftime('%S')}")
        await asyncio.sleep(0.1)

async def main():
    hb = asyncio.create_task(heartbeat())
    await asyncio.sleep(0.3)
    print("--- calling register_bad ---")
    await register_bad("x")             # heartbeat stops during this
    print("--- calling register_good ---")
    await register_good("x")            # heartbeat keeps ticking
    await asyncio.sleep(0.3)
    hb.cancel()

asyncio.run(main())
```

The heartbeat is the instrument. Watch where the ticks stop — that gap is what
every other connection experienced.

Now a snippet that claims to fix a blocking loop. **Find the fault before
reading on:**

```javascript
// "Chunked so it doesn't block" — processing 1M records
async function processAll(records) {
  const out = [];
  for (const r of records) {
    out.push(expensiveTransform(r));     // ~0.05ms each
    await Promise.resolve();             // yield back to the loop?
  }
  return out;
}
```

*Answer.* `await Promise.resolve()` schedules a **microtask**, and the microtask
queue is drained to exhaustion before the loop advances to its I/O phase. So
every iteration yields to other microtasks and to nothing else: no timer fires,
no socket is read, for the entire million records. The loop is blocked exactly as
before, and now with a million extra queue operations. `await new Promise(r =>
setImmediate(r))` yields to the check phase and does let I/O through — but
yielding per item is far too often. Batch: yield every few thousand records, or
move the whole job off the request path.

Python has the same trap: `await asyncio.sleep(0)` does yield to the loop, but
per-item it costs more than the work.

## Tradeoffs & when it's wrong

**Offloading to a thread** is right for blocking I/O and for bounded CPU work.
It costs a bounded pool that can itself saturate — and in CPython, a thread does
not help pure-Python CPU work at all, because of the GIL.

**Offloading to a process or worker thread** is right for real CPU work. It costs
serialisation of the arguments and results, so it is wrong when the data is large
relative to the computation. Sending 5 MB to a worker to save 3 ms is a loss.

**Chunking with periodic yields** keeps the work in-process and is right when the
work must stay on the request path and is modest. It is wrong for genuinely large
jobs: you are still consuming the whole worker's capacity, just politely, and the
total latency is unchanged.

**Moving the work to a queue** is right when the caller does not need the result
now, and is the only option that actually removes the load. It costs an
architecture: a broker, a worker fleet, and a way for the client to collect the
result later.

**Adding more workers is not a fix.** N workers each blockable by one bad request
means it takes N concurrent bad requests to stall the service instead of one.
That raises the bar without changing the failure.

What would change the choice: how large the work is, whether the result is needed
synchronously, and whether the input size is attacker-controlled. That last one
turns a performance question into a denial-of-service question.

## Failure modes & operational cost

- **Misattributed latency.** The dominant cost. Time is charged to the victim,
  so investigations start in the wrong service.
- **Health check flapping.** A blocked worker fails its liveness probe, gets
  killed, and its in-flight requests are lost — converting a latency problem into
  an availability one.
- **Timeout cascades.** Callers time out and retry, adding load to a worker that
  is already stalled. Without retry budgets this becomes a retry storm.
- **Unbounded input as a DoS vector.** If a 10 MB body means a 400 ms stall, the
  request rate needed to take the service down is small and cheap.
- **Invisible thread-pool saturation.** In Node, more than `UV_THREADPOOL_SIZE`
  concurrent `fs`/`zlib`/`pbkdf2` calls queue with no CPU saturation and no error.
- **Metrics that cannot see it.** Per-endpoint latency alone will not reveal the
  cause. Loop delay is the metric that names the culprit, and almost nobody
  exports it until after their first incident.

## Open questions / to verify

Not checked against primary docs in this session:

- Python's default `loop.slow_callback_duration` (believed 0.1s) and whether
  `set_debug(True)` carries other overhead unsuitable for production.
- Whether `monitorEventLoopDelay` is stable API in the Node version in use, and
  what `resolution` costs.
- Whether `asyncio.to_thread` and `loop.run_in_executor(None, ...)` share the same
  default pool, and that pool's size.
- Whether any JSON parser in either runtime yields during parsing, or whether all
  of them block for the full input.
- Whether `py-spy dump` requires elevated privileges in the container runtime used
  here.

Candidate for `practiced`: export loop-delay as a metric from a real service, add
a deliberate 200 ms synchronous stretch in one endpoint, and confirm you can
identify the culprit endpoint *from the metrics alone* — without knowing in
advance where you put it.

## Sources

- [Node — Don't block the event loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)
- [Node `perf_hooks.monitorEventLoopDelay`](https://nodejs.org/api/perf_hooks.html)
- [Python — Developing with asyncio (debug mode)](https://docs.python.org/3/library/asyncio-dev.html)
- [py-spy](https://github.com/benfred/py-spy)
