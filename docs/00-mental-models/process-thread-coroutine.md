---
title: Process vs thread vs coroutine
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

Three units of execution, differing in what they isolate, who schedules them, and
what they cost. Choosing among them is the central structural decision in
[server architecture](wsgi-asgi-and-clustering.md), and the reason the same code
scales in one runtime and collapses in another.

**Process.** Its own virtual address space, its own file descriptor table, its
own interpreter. Scheduled preemptively by the OS. Two processes cannot corrupt
each other's memory, so a crash kills one and leaves the rest running. Sharing
anything requires serialising it through a pipe, socket, or shared memory
segment.

**Thread.** Lives inside a process and shares its address space and file
descriptors with every sibling. Also scheduled preemptively by the OS, which
means it can be interrupted **between any two machine instructions**. That is
what makes shared mutable state dangerous: you never chose the interruption
point, so you have to defend every place it could happen.

**Coroutine** (task, green thread, goroutine-like). A function that can suspend
and resume, scheduled in user space by an event loop rather than by the kernel.
It yields **only at points you wrote** — in Python and JavaScript, at `await`.
Between two `await`s, a coroutine holds the thread exclusively and cannot be
preempted.

That last property is the whole trade. Cooperative scheduling means no lock is
needed to protect a sequence with no `await` in it, because nothing else can run.
It also means a coroutine that never yields — a long computation, or a blocking
call — starves every other task on that loop.

### Concurrency vs parallelism

These are different questions and the words are not interchangeable.

**Concurrency is a structuring property:** several tasks are in progress, and the
system interleaves them. **Parallelism is an execution property:** several things
run at the same instant, which requires several cores.

- Concurrent, not parallel: a single-threaded event loop holding 10,000 open
  sockets. One core, one instruction at a time, 10,000 conversations in flight.
- Parallel, and barely concurrent: four processes each grinding one CPU-bound
  computation to completion.
- Both: four worker processes, each running an event loop with thousands of
  sockets. This is what most production Python and Node deployments actually are.

The distinction has a practical consequence: **concurrency is how you stop
waiting; parallelism is how you do more work.** If your requests are waiting
(see [CPU-bound vs I/O-bound](cpu-bound-vs-io-bound.md)), concurrency wins and
extra cores buy little. If they are computing, only parallelism helps, and no
amount of `async` will.

## Why it exists (what came before)

Processes came first, because isolation was the point — a multi-user system must
not let one program read or corrupt another's memory. Early servers forked a
process per connection. Correct, and expensive: CGI's fork-per-request cost is
what shaped a generation of "keep the process alive" designs.

Threads were added because that isolation is sometimes exactly what you don't
want. Sharing an address space makes communication free, and creation cheaper
than a fork. The bill arrives as locks, races, and deadlocks — every problem the
address-space boundary had been solving for you.

Coroutines came from a different pressure: the C10k problem. A thread carries a
kernel-allocated stack, so tens of thousands of threads cost real memory and real
scheduler work, and most of those threads are only *waiting on a socket*. If
waiting is the common case, waiting should be cheap — so move scheduling into
user space, make the unit of work a suspended function rather than a stack, and
let one thread hold many waits.

## Smallest example

Three shapes of the same "do two slow things" problem.

```python
import asyncio, threading, time
from concurrent.futures import ProcessPoolExecutor

def blocking_io():                       # a real blocking call
    time.sleep(1)
    return "io done"

def cpu_work(n):                         # burns cycles, never waits
    return sum(i * i for i in range(n))

# 1. Threads: two blocking calls overlap, because sleep releases the GIL
def with_threads():
    t0 = time.perf_counter()
    ts = [threading.Thread(target=blocking_io) for _ in range(2)]
    [t.start() for t in ts]
    [t.join() for t in ts]
    print(f"threads: {time.perf_counter() - t0:.2f}s")   # ~1s, not ~2s

# 2. Coroutines: same overlap, no threads, explicit yield points
async def with_coroutines():
    t0 = time.perf_counter()
    await asyncio.gather(asyncio.sleep(1), asyncio.sleep(1))
    print(f"coroutines: {time.perf_counter() - t0:.2f}s")  # ~1s

# 3. Processes: the only shape that gives CPU parallelism in CPython
def with_processes():
    t0 = time.perf_counter()
    with ProcessPoolExecutor(max_workers=2) as pool:
        list(pool.map(cpu_work, [5_000_000, 5_000_000]))
    print(f"processes: {time.perf_counter() - t0:.2f}s")

if __name__ == "__main__":
    with_threads()
    asyncio.run(with_coroutines())
    with_processes()
```

```typescript
import { Worker } from "node:worker_threads";

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

// Coroutine-shaped: two waits overlap on one thread
async function withPromises() {
  const t0 = performance.now();
  await Promise.all([sleep(1000), sleep(1000)]);
  console.log(`promises: ${((performance.now() - t0) / 1000).toFixed(2)}s`);
}

// Parallel: a second thread, because CPU work cannot share the first
function withWorker(): Promise<number> {
  const src = `
    const { parentPort } = require("node:worker_threads");
    let s = 0;
    for (let i = 0; i < 5_000_000; i++) s += i * i;
    parentPort.postMessage(s);
  `;
  return new Promise((resolve, reject) => {
    const w = new Worker(src, { eval: true });
    w.once("message", resolve);
    w.once("error", reject);
  });
}

await withPromises();
await Promise.all([withWorker(), withWorker()]);
```

The instructive part is what each cannot do. Swap `asyncio.sleep(1)` for
`time.sleep(1)` inside the coroutine version and the total becomes ~2s — you
blocked the loop. Run `cpu_work` in two threads instead of two processes and
CPython gives you no speedup at all, for reasons in
[Python's execution model](python-execution-model.md).

## Tradeoffs & when it's wrong

**Cost model, roughly, and worth re-measuring on your own hardware rather than
trusting these magnitudes:**

| | Isolation | Scheduler | Memory per unit | Practical count |
|---|---|---|---|---|
| Process | Full address space | Kernel, preemptive | MBs | tens |
| Thread | None within a process | Kernel, preemptive | Stack, often ~1–8 MB reserved | thousands |
| Coroutine | None | User space, cooperative | KBs | 100k+ |

**Processes** are right when you need crash isolation, real CPU parallelism, or
to escape a global interpreter lock. They are wrong when tasks must share large
mutable state, because you will pay serialisation on every exchange and
eventually build a worse database.

**Threads** are right for blocking calls you do not control — a driver with no
async version, a C library, a filesystem API. They are wrong as a general
concurrency strategy at high connection counts, and they are wrong whenever the
shared state is subtle, because preemption means the bug appears once a week
under load and never in a test.

**Coroutines** are right when the workload is overwhelmingly waiting and you
control the libraries, so every blocking call has an async equivalent. They are
wrong when any significant CPU work sits in the request path, and wrong when a
key dependency is synchronous — one blocking driver call inside a coroutine
stalls every other connection on that loop, and the symptom is *global* latency
rather than a slow endpoint, which makes it maddening to attribute.

What would change the recommendation: the shape of the work, not taste. Measure
where time goes first; the answer picks the primitive.

## Failure modes & operational cost

- **Race conditions** (threads). Two threads read-modify-write the same value and
  one update vanishes. Preemption can happen between any two instructions, so
  "it's just an increment" is not a defence.
- **Deadlock** (threads, processes). Two holders each wait for the other's lock.
  Consistent lock ordering prevents it; timeouts make it survivable.
- **Starving the event loop** (coroutines). One long synchronous stretch and
  every connection on that loop waits. Covered in
  [blocking the event loop](blocking-the-event-loop.md).
- **Thread pool exhaustion.** A bounded pool with every thread parked on a slow
  call looks identical to a hung process from outside, and the queue behind it is
  invisible unless you instrumented it.
- **Fork after threads.** Forking a process that holds threads copies the memory
  but not the threads; a lock held at fork time stays held forever in the child.
  This is why start method matters in
  [Python's execution model](python-execution-model.md).
- **Per-unit state that isn't shared.** An in-process cache in a four-worker
  deployment is four caches with four different answers. Correct behaviour in
  development, incoherent in production.

Operationally: processes cost memory and make shared state a network problem;
threads cost correctness attention; coroutines cost library discipline — one
synchronous dependency undoes the model.

## Open questions / to verify

Not checked against primary sources in this session:

- Default thread stack size on this platform, and how much is reserved versus
  actually committed. The table above gives a range because it varies by OS and
  by whether the stack is touched.
- Actual per-task memory of a Python `asyncio` task and a Node promise chain,
  measured rather than assumed.
- Whether the "concurrency is dealing with many things, parallelism is doing many
  things" phrasing is Rob Pike's exactly, and where he said it.
- Cost of a kernel context switch versus a coroutine switch on this machine,
  measured with a benchmark rather than quoted.

Candidate for `practiced`: write one script that serves N concurrent slow
requests three ways — threads, coroutines, processes — and plot throughput as N
rises from 10 to 10,000. The crossover points are the lesson.

## Sources

- [Python `threading`](https://docs.python.org/3/library/threading.html),
  [`asyncio`](https://docs.python.org/3/library/asyncio.html),
  [`multiprocessing`](https://docs.python.org/3/library/multiprocessing.html)
- [Node `worker_threads`](https://nodejs.org/api/worker_threads.html)
- [The C10k problem](http://www.kegel.com/c10k.html) — the historical framing for
  why coroutines exist
