---
title: CPU-bound vs I/O-bound work
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

A classification of where a unit of work spends its time, and therefore which
scaling lever moves it.

**I/O-bound** work spends most of its wall-clock time waiting for something
external: a database, an HTTP call, a disk read. The CPU is idle during the wait.
Almost every CRUD endpoint is I/O-bound — it waits on a query, formats the
result, returns.

**CPU-bound** work spends its time executing instructions. Nothing is being
waited on; the core is busy. Hashing a password, compressing a payload,
serialising a large object, rendering a template, evaluating a regex.

The reason this classification earns a place in mental models rather than
trivia: **it determines whether concurrency or parallelism is the answer**, and
the two are not interchangeable (see
[process vs thread vs coroutine](process-thread-coroutine.md)).

- I/O-bound: one thread can hold thousands of waits. Concurrency multiplies
  throughput enormously; extra cores do almost nothing.
- CPU-bound: concurrency multiplies *nothing*. A busy core interleaved among ten
  tasks finishes the same total work, only later and with switching overhead.
  Only more cores — or less work — help.

Get this backwards and you scale the wrong axis: adding `async` to a
CPU-bound service, or adding cores to a service that is waiting on one slow
database.

### The classification is per operation, not per service

This is where real systems get interesting. A service is rarely uniformly one or
the other, and the CPU-bound part hides inside code that looks like I/O:

- Serialising a 5 MB JSON response — CPU-bound, in the middle of an "I/O" handler
- `bcrypt`/`argon2` password verification — deliberately CPU-bound, by design
- gzip on the way out, TLS handshakes, image resizing
- A regex with catastrophic backtracking — CPU-bound and unbounded

A handler that is 5 ms of query wait and 200 ms of JSON serialisation is a
CPU-bound handler wearing I/O clothing, and every architectural instinct you
apply to it will be wrong until you measure.

## Why it exists (what came before)

The distinction is as old as timesharing. Early operating systems discovered that
a program blocked on tape or disk left the CPU idle, so the scheduler could run
something else meanwhile — the birth of multiprogramming. The classification is
just naming which of the two resources a job contends for.

Before async I/O was widely available, the standard answer to I/O-bound
concurrency was a thread or process per connection, because a blocked thread was
the only cheap way to hold a wait. That works until the connection count makes
stacks and scheduling expensive — the C10k problem — after which the industry
moved waits into user space. What did not change is the underlying fact: waiting
and computing are different resources, and no scheduler trick converts one into
the other.

## Smallest example

The measurement, not the theory. Both scripts run the same two workloads
concurrently and report wall-clock time.

```python
import asyncio, time

def cpu_task(n=10_000_000):
    total = 0
    for i in range(n):          # deliberately naive; this is the point
        total += i * i
    return total

async def io_task():
    await asyncio.sleep(1)      # stands in for a query or HTTP call

async def main():
    # I/O-bound: 4 tasks, each waiting 1s -> ~1s total. Concurrency works.
    t0 = time.perf_counter()
    await asyncio.gather(*(io_task() for _ in range(4)))
    print(f"4x I/O concurrently: {time.perf_counter() - t0:.2f}s")

    # CPU-bound on the same loop: 2 tasks -> ~sum of both. Concurrency does nothing.
    t0 = time.perf_counter()
    await asyncio.gather(
        asyncio.to_thread(cpu_task),   # threads don't help either, in CPython
        asyncio.to_thread(cpu_task),
    )
    print(f"2x CPU 'concurrently': {time.perf_counter() - t0:.2f}s")

    t0 = time.perf_counter()
    cpu_task(); cpu_task()
    print(f"2x CPU sequentially:   {time.perf_counter() - t0:.2f}s")

asyncio.run(main())
```

Run it. The two CPU numbers will be close to each other, and the I/O number will
be near 1s rather than 4s. That contrast is the entire topic.

```typescript
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

function cpuTask(n = 10_000_000): number {
  let total = 0;
  for (let i = 0; i < n; i++) total += i * i;
  return total;
}

// I/O-bound: four waits overlap on one thread
let t0 = performance.now();
await Promise.all([sleep(1000), sleep(1000), sleep(1000), sleep(1000)]);
console.log(`4x I/O concurrently: ${((performance.now() - t0) / 1000).toFixed(2)}s`);

// CPU-bound: Promise.all does NOT make these overlap — there is one thread
t0 = performance.now();
await Promise.all([
  Promise.resolve().then(() => cpuTask()),
  Promise.resolve().then(() => cpuTask()),
]);
console.log(`2x CPU "concurrently": ${((performance.now() - t0) / 1000).toFixed(2)}s`);
```

The Node version makes a sharper point: wrapping CPU work in a promise makes it
*look* concurrent and changes nothing. `Promise.all` expresses "these may
overlap," not "run these on different cores." Nothing overlaps, because there is
one thread — see [Node's execution model](node-execution-model.md).

## How to tell which one you have

Measure; do not reason from the endpoint's name.

1. **Compare wall-clock time against CPU time for the same operation.** If wall
   time is 200 ms and CPU time is 5 ms, you waited. If they are close, you
   computed. In Python, `time.perf_counter()` versus `time.process_time()` gives
   you exactly this pair.
2. **Watch utilisation under load.** Cores pinned near 100% with modest
   throughput means CPU-bound. Low CPU with rising latency means you are waiting
   — on a dependency, a lock, a connection pool, or a queue.
3. **Profile rather than guess.** `py-spy` and Node's `--prof` show where time
   actually goes; the surprise is usually serialisation, logging, or crypto.
4. **Check the tail, not the mean.** A handler that is I/O-bound at p50 can be
   CPU-bound at p99 because that is where the large payloads are.

## Tradeoffs & when it's wrong

**Treating everything as I/O-bound** is the default failure of adopting async.
The framework is async, so the service must be — until a CPU-bound stretch in one
handler adds latency to every connection on the loop, and the metric that moves
is unrelated to the endpoint that caused it.

**Treating everything as CPU-bound** produces the opposite waste: worker counts
tuned to core counts on a service whose threads are all parked on a database,
where ten times the workers would have served ten times the traffic on the same
hardware.

The honest position is that most services are mixed, and the useful engineering
move is to *separate* them: keep the request path I/O-bound and push CPU-bound
work off it — a thread pool, a worker process, or a queue.

What would change the answer: measurement. Also payload size, which can flip a
handler's classification without a line of code changing.

## Failure modes & operational cost

- **CPU work on an event loop.** Global latency rise, attributed to the wrong
  endpoint. See [blocking the event loop](blocking-the-event-loop.md).
- **More workers for CPU-bound work.** Beyond core count you add context
  switching and memory, not throughput. Throughput plateaus and then declines.
- **More workers for I/O-bound work, without checking downstream limits.** Each
  worker holds database connections; the pool multiplies by worker count and
  exhausts the server's connection limit. The bottleneck moves rather than
  disappears.
- **Async wrappers around blocking calls.** `async def` around a synchronous
  driver call does not make it async; it makes it a blocking call the loop cannot
  see.
- **Measuring with the wrong clock.** Wall-clock time alone cannot distinguish
  the two cases; that is precisely what CPU time is for.
- **Utilisation as a target.** A queueing system's latency rises non-linearly as
  utilisation approaches 1, so "the cores are 95% busy, good" is usually a
  latency problem in progress — the reasoning lives in
  [queueing theory](../04-caching-and-performance/queueing-theory.md).

## Open questions / to verify

- Whether `time.process_time()` on this platform counts CPU time for the calling
  thread only or the whole process, which changes how to read the comparison in a
  threaded program.
- Whether `asyncio.to_thread` for CPU-bound work is ever faster than sequential
  in CPython for pure-Python loops — expected not, because of the GIL, but the
  script above will show it directly.
- What fraction of a representative JSON response's handling is serialisation, on
  this hardware, at 10 KB / 100 KB / 5 MB payload sizes.
- Whether Node's JSON parsing releases the loop at any point for large inputs, or
  blocks for the whole parse.

Candidate for `practiced`: take the naive `cpu_task` handler, measure p50/p99
under concurrent load, move it to a worker, and measure again. The number to
report is the change in *unrelated* endpoints' latency.

## Sources

- [Python `time` — `perf_counter` vs `process_time`](https://docs.python.org/3/library/time.html)
- [Python `asyncio` — running blocking code](https://docs.python.org/3/library/asyncio-dev.html)
- [Node `worker_threads`](https://nodejs.org/api/worker_threads.html)
