---
title: Python's execution model
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

How CPython actually runs your code: one lock around bytecode execution, an
optional single-threaded event loop, and a separate-process escape hatch.

This topic has no TypeScript counterpart — the equivalent material is
[Node's execution model](node-execution-model.md). The two runtimes reach similar
places by different routes, and the differences are where the bugs live.

### The GIL

The Global Interpreter Lock is a single mutex that a thread must hold to execute
Python bytecode. One thread runs Python at a time, per interpreter.

It exists to protect interpreter internals — reference counts above all. Every
object carries a refcount that changes constantly, and making each of those
changes individually atomic would cost more than the lock does.

The two consequences that matter:

1. **Threads give no CPU parallelism for pure-Python code.** Two threads summing
   integers take about as long as doing it sequentially, plus switching overhead.
2. **Threads still help enormously for I/O.** The GIL is *released* around
   blocking calls — socket reads, file operations, `time.sleep` — so while one
   thread waits on the network, another runs Python. A thread parked in `recv()`
   holds no lock.

So "the GIL makes Python threads useless" is wrong. It makes them useless for
computation and entirely appropriate for waiting.

C extensions may also release the GIL explicitly around their own work, which is
why NumPy array operations and many compression and crypto libraries do achieve
real parallelism from threads. The parallelism is in the C code, not in Python.

### asyncio

A single-threaded event loop with cooperative scheduling. Coroutines run until
they hit `await`, then yield control back to the loop, which runs whatever is
ready next.

The mental model that prevents most asyncio bugs: **`await` is the only place
another task can run.** Between two `await`s your coroutine holds the thread
absolutely — which is why you rarely need locks, and why one slow synchronous
line stalls the entire process.

Because it is one thread, `asyncio` gives concurrency and never parallelism.
Its purpose is holding many waits cheaply, not doing more work at once.

Escape hatches for code that would block the loop:

- `asyncio.to_thread(fn, *args)` — run a blocking call in the default thread
  pool. Correct for I/O-bound blocking calls, useless for CPU-bound Python.
- `loop.run_in_executor(pool, fn)` — the same idea with a pool you control,
  including a `ProcessPoolExecutor` for CPU-bound work.

### multiprocessing

Separate processes, therefore separate interpreters, therefore separate GILs and
real CPU parallelism. Arguments and return values are pickled and shipped, which
bounds how chatty the work can profitably be.

Start methods differ in what the child inherits, and the difference is not
cosmetic:

- **fork** — child is a memory copy of the parent. Fast to start, inherits
  everything, and **unsafe in a process that has threads**: locks held at fork
  time remain held forever in the child, and the thread that would have released
  them does not exist there. This is a real source of hangs in threaded servers
  that fork workers.
- **spawn** — child starts a fresh interpreter and re-imports your module. Slower,
  safe, and requires that everything you pass be picklable and that top-level
  code be guarded by `if __name__ == "__main__":`.
- **forkserver** — fork from a small clean server process, avoiding the
  threads-at-fork hazard while keeping most of fork's speed.

## Why it exists (what came before)

CPython chose the GIL early, when machines were single-core and the alternative
was fine-grained locking on every object — which is slower for single-threaded
code, and single-threaded code was the overwhelming majority. It was the right
trade for the hardware of the time, and it has been extremely difficult to remove
because the entire C extension ecosystem was written assuming it.

`asyncio` arrived late (3.4, with `async`/`await` syntax in 3.5) after a decade of
third-party event loops — Twisted, Tornado, gevent — proved the model. Before it,
Python's answer to many connections was threads, processes, or greenlets
monkey-patching the standard library.

Free-threaded CPython (PEP 703) is the long-running attempt to remove the GIL
while keeping the ecosystem, using biased reference counting and deferred
reclamation instead of one big lock.

## Smallest example

```python
import asyncio, threading, time
from concurrent.futures import ProcessPoolExecutor

def cpu(n=8_000_000):
    t = 0
    for i in range(n):
        t += i * i
    return t

def timed(label, fn):
    t0 = time.perf_counter()
    fn()
    print(f"{label:28} {time.perf_counter() - t0:.2f}s")

def two_threads_cpu():                       # GIL: no speedup expected
    ts = [threading.Thread(target=cpu) for _ in range(2)]
    [t.start() for t in ts]; [t.join() for t in ts]

def two_threads_io():                        # GIL released: real overlap
    ts = [threading.Thread(target=lambda: time.sleep(1)) for _ in range(2)]
    [t.start() for t in ts]; [t.join() for t in ts]

def two_procs_cpu():                         # separate GILs: real parallelism
    with ProcessPoolExecutor(max_workers=2) as p:
        list(p.map(cpu, [8_000_000, 8_000_000]))

async def blocking_in_coroutine():           # the classic mistake
    t0 = time.perf_counter()
    await asyncio.gather(
        asyncio.to_thread(time.sleep, 1),    # correct: pushed to a thread
        asyncio.to_thread(time.sleep, 1),
    )
    print(f"{'blocking via to_thread':28} {time.perf_counter() - t0:.2f}s")

    t0 = time.perf_counter()
    async def bad():
        time.sleep(1)                        # WRONG: blocks the whole loop
    await asyncio.gather(bad(), bad())
    print(f"{'blocking inside coroutine':28} {time.perf_counter() - t0:.2f}s")

if __name__ == "__main__":
    timed("sequential CPU x2", lambda: (cpu(), cpu()))
    timed("2 threads, CPU", two_threads_cpu)
    timed("2 threads, I/O", two_threads_io)
    timed("2 processes, CPU", two_procs_cpu)
    asyncio.run(blocking_in_coroutine())
```

Expected shape of the output, and worth confirming rather than trusting: threaded
CPU ≈ sequential CPU; threaded I/O ≈ 1s; processes ≈ half of sequential;
`to_thread` ≈ 1s; blocking inside a coroutine ≈ 2s. That last pair is the same
work with the same total sleep, differing only in whether the loop could see it.

## Tradeoffs & when it's wrong

**asyncio** is right when the workload is dominated by waiting and every
dependency has an async driver. It is wrong when a critical library is
synchronous only, and wrong when CPU work sits in the request path — the cost is
paid by *every* connection, not by the slow endpoint, which makes attribution
hard.

**Threads** are right for blocking calls you cannot avoid, and for C extensions
that release the GIL. They are wrong as a CPU parallelism strategy, and their
shared-state hazards are real: the GIL makes individual bytecode steps atomic but
guarantees nothing about `x += 1`, which is several steps.

**Processes** are right for CPU-bound work and for isolation. They are wrong when
data must move constantly, because pickling dominates, and they multiply memory
and downstream connection counts.

The argument against async as a default deserves stating, since the ecosystem
assumes it: sync code with threads is easier to reason about, easier to profile,
and cannot be sabotaged by one blocking call. If a service is I/O-bound but
modest in connection count, a threaded WSGI deployment is simpler and fast
enough. What would change that: connection count, long-lived connections
(WebSockets, SSE), or a fan-out pattern where one request awaits many
dependencies — all of which favour async decisively.

## Failure modes & operational cost

- **A blocking call inside `async def`.** The single most common asyncio bug.
  `requests` instead of `httpx`, a synchronous DB driver, `time.sleep`,
  `open().read()` of a large file. Symptom: global latency, unrelated endpoints
  slow.
- **CPU work inside a coroutine.** Same symptom, no fix except moving it out.
- **Fork after threads.** Hangs that reproduce only under load, because the
  inherited lock state depends on timing.
- **Unbounded task creation.** `asyncio.create_task` in a loop with no limit
  builds an unbounded queue; memory grows and latency degrades with no error.
- **Fire-and-forget tasks.** A task whose reference is dropped may be garbage
  collected, and exceptions inside a never-awaited task can vanish silently.
- **Default thread pool sizing.** `to_thread` shares one bounded default pool; a
  handful of slow calls can starve everything else that uses it.
- **Assuming `x += 1` is atomic.** It is not; the GIL can be released between the
  read and the write.

## Open questions / to verify

All of these are version-specific and were not checked against primary docs here:

- The default GIL switch interval (`sys.getswitchinterval()`) on this build.
- The default `ThreadPoolExecutor` worker count used by `asyncio.to_thread` in
  Python 3.13, and whether the `min(32, cpu + 4)` formula still holds.
- The default `multiprocessing` start method on Linux for 3.13 and 3.14, since
  there has been movement away from `fork` — confirm before relying on either.
- Current status of free-threaded builds (PEP 703): which versions ship it, how
  it is enabled, and whether it is still labelled experimental.
- Precisely which standard-library calls release the GIL. The general rule is
  "blocking I/O does," but the exact list matters when diagnosing a stall.
- Whether `uvloop` still gives a material speedup over the stdlib loop on current
  versions, and for what workload shape.

Candidate for `practiced`: take an `asyncio` service, add one synchronous
database call in one endpoint, and measure the latency of a *different* endpoint
under load before and after. Then fix it with `to_thread` and measure again.

## Sources

- [Python `asyncio`](https://docs.python.org/3/library/asyncio.html) and
  [Developing with asyncio](https://docs.python.org/3/library/asyncio-dev.html)
- [`multiprocessing` — start methods](https://docs.python.org/3/library/multiprocessing.html#contexts-and-start-methods)
- [PEP 703 — Making the GIL optional](https://peps.python.org/pep-0703/)
- [PEP 492 — `async`/`await` syntax](https://peps.python.org/pep-0492/)
