---
title: Sockets and file descriptors
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

A file descriptor is a small non-negative integer: an index into a table the
kernel keeps for your process. Each entry points at something the kernel can read
from or write to.

The unifying idea in Unix is that almost everything is one of these. An open
file, a TCP socket, a pipe, a Unix domain socket, an `epoll` instance, a timer, a
signal handler — all file descriptors, all drawn from the same per-process table,
all counted against the same limit.

That shared accounting is the whole reason this topic exists. A service that
leaks database connections and a service that leaks open log files fail the same
way, with the same error, because they are exhausting the same table.

Descriptors 0, 1, and 2 are stdin, stdout, stderr by convention. Everything your
process opens after that gets the lowest available number, which is why a leak
shows up as a steadily climbing count.

### Why the limit exists and where it comes from

Three limits stack, and they are commonly confused:

1. **Per-process soft limit** (`RLIMIT_NOFILE`, what `ulimit -n` prints). The one
   you actually hit. A process may raise its own soft limit up to the hard limit
   without privileges.
2. **Per-process hard limit.** The ceiling on the soft limit. Raising it needs
   privileges or a service-manager directive (`LimitNOFILE=` in systemd).
3. **System-wide** (`fs.file-max`, and `fs.nr_open` as the per-process ceiling).
   Rarely the binding constraint on a modern machine.

Containers matter here: the limit your process sees comes from the container
runtime's configuration, not the host's, and it is frequently lower than you
expect. A service that runs fine on a laptop with a limit of 1,048,576 can fail
in a pod with 1,024.

### What exhaustion looks like

The error is `EMFILE` — "too many open files" — and where it surfaces determines
how bad it is:

- On `accept()`: **the server stops accepting connections but the process stays
  alive.** Health checks that only test the process, or that reuse an existing
  connection, keep passing. Clients see connection timeouts. This is the worst
  variant because every layer reports "up".
- On `open()`: file operations fail, often in logging, which then hides the
  errors that would have told you what happened.
- On `connect()`: outbound calls fail, so the service looks like it has a
  dependency problem rather than a resource problem.

## Why it exists (what came before)

The design is from early Unix: rather than separate APIs per resource kind, give
every kind the same integer handle and the same `read`/`write`/`close` verbs. It
is why `epoll` can watch sockets, pipes, and timers with one mechanism, and why
shell redirection works uniformly.

The limit exists because the kernel allocates memory per descriptor and per
socket buffer, and because a per-process table with no bound lets one process
exhaust a shared machine. The default is deliberately conservative — it was set
when servers held tens of connections, and it has been raised repeatedly without
ever becoming generous, because the correct value depends entirely on the
workload.

## Smallest example

A leak, and the same code without the leak.

```python
import socket

# LEAKS: on any exception between connect and close, the fd is never released.
def fetch_bad(host, port):
    s = socket.create_connection((host, port), timeout=2)
    s.sendall(b"GET / HTTP/1.0\r\n\r\n")
    data = s.recv(4096)          # raises on timeout -> close() never runs
    s.close()
    return data

# CORRECT: the context manager closes on every path, including exceptions.
def fetch_good(host, port):
    with socket.create_connection((host, port), timeout=2) as s:
        s.sendall(b"GET / HTTP/1.0\r\n\r\n")
        return s.recv(4096)
```

The bad version is not obviously wrong, which is the point: it closes the socket
on the happy path, and the happy path is what tests exercise. Timeouts and
connection resets are exactly the conditions under load, so the leak rate rises
with traffic — the service degrades fastest when it is busiest.

Watch it happen. This counts your own descriptors from inside the process:

```python
import os

def fd_count():
    return len(os.listdir(f"/proc/{os.getpid()}/fd"))   # Linux
```

```javascript
// Node: the fd table is shared by fs handles and sockets alike.
import { readdirSync } from "node:fs";

const fdCount = () => readdirSync(`/proc/${process.pid}/fd`).length;

// An HTTP agent with keep-alive holds sockets open on purpose.
// maxSockets is per host; total held = maxSockets x hosts x processes.
import { Agent } from "node:http";
const agent = new Agent({ keepAlive: true, maxSockets: 50 });
```

That last comment is where real exhaustion usually comes from — not a leak at
all, but a correctly-working pool sized without multiplying by host count and
worker count.

## Diagnosing it

```bash
cat /proc/<pid>/limits | grep 'open files'   # the limits this process actually has
ls /proc/<pid>/fd | wc -l                    # how many it is using now
ls -l /proc/<pid>/fd | head -20              # what they are: sockets, files, pipes
lsof -p <pid> | awk '{print $5}' | sort | uniq -c   # grouped by type
ss -tanp | awk '{print $1}' | sort | uniq -c        # socket states, system-wide
```

The distribution over socket **state** is the diagnostic that identifies the bug,
and two states are routinely mixed up:

| State | Meaning | Whose bug |
|---|---|---|
| `CLOSE_WAIT` | The peer sent FIN; **your** application has not called `close()` | Yours — a leak, almost always |
| `TIME_WAIT` | You closed actively; the kernel holds the tuple briefly | Nobody's — normal, though volume may indicate no connection reuse |

Piles of `CLOSE_WAIT` mean your code is holding descriptors for connections the
other side has already finished with. Piles of `TIME_WAIT` mean you are opening
and closing many short connections — a keep-alive problem, not a leak.

## Tradeoffs & when it's wrong

**Raising the limit** is right when the workload genuinely needs many concurrent
connections — a proxy, a WebSocket gateway, an event-loop server holding tens of
thousands of sockets. It is wrong as a response to an unexplained climb: a leak
with a higher ceiling is the same leak with a longer fuse and a less convenient
failure time.

The honest test: does the count plateau under steady load, or climb without
bound? Plateauing high means size the limit. Climbing means find the leak.

**Aggressive keep-alive pooling** is right because it removes handshake cost —
2–3 round trips per request, per [the request
lifecycle](request-lifecycle.md). It costs one held descriptor per idle
connection, multiplied by hosts and by worker count, and the multiplication is
where deployments get surprised.

**Closing connections eagerly** frees descriptors and costs handshakes, and
strands ports in `TIME_WAIT` on your side at volume — see
[TCP fundamentals](tcp-fundamentals.md).

What would change the answer: connection lifetime and fan-out. A service calling
three backends with keep-alive needs a very different limit from one calling
three hundred.

## Failure modes & operational cost

- **Silent accept failure.** The process is alive, the port is bound, and no new
  connection can be served. Liveness probes that check the process, or that reuse
  a pooled connection, will not notice.
- **Leak on the error path only.** Passes every test, fails under load, because
  the leak needs a timeout or a reset to trigger.
- **Pool size multiplied by workers.** Sixteen workers x 50 sockets per host x 5
  hosts is 4,000 descriptors before the first file is opened.
- **Logging failures masking the cause.** Exhaustion breaks `open()`, so the log
  line explaining the outage cannot be written.
- **`CLOSE_WAIT` accumulation from an unclosed response body.** In many HTTP
  clients the connection returns to the pool only when the body is fully read or
  explicitly closed; reading the status and discarding the response leaks.
- **Descriptors held by zombie children.** A forked worker that exited without
  being reaped can keep inherited descriptors alive.
- **Inherited descriptors across `fork()`.** Children share the parent's open
  sockets. A connection created before fork is the *same* connection in every
  worker — see [WSGI vs ASGI and clustering](wsgi-asgi-and-clustering.md).

Operationally: descriptor count is a first-class metric, and its *derivative*
matters more than its value. Alert on sustained growth, not on a threshold.

## Open questions / to verify

Not checked against primary sources in this session:

- The actual soft and hard `RLIMIT_NOFILE` for this machine, and separately for
  the container runtime used in deployment.
- Whether Node's default `http.Agent` has `keepAlive` on by default in the
  version in use, and its default `maxSockets`.
- Whether Python's `http.client`/`requests`/`httpx` return a pooled connection on
  response-object garbage collection or require explicit close, per library.
- Whether `/proc/<pid>/fd` counting includes `epoll` and eventfd descriptors — it
  should, but confirm before using the count as a connection estimate.
- The default `TIME_WAIT` duration on this kernel and whether it is tunable
  separately from `tcp_fin_timeout`.

Candidate for `practiced`: run a service with a deliberately low `ulimit -n`,
drive it until `accept()` fails, and confirm from the outside that the process
still looks healthy. Then add a descriptor-count metric and an alert on its slope,
and verify the alert fires before the failure.

## Sources

- [`getrlimit(2)` / `RLIMIT_NOFILE`](https://man7.org/linux/man-pages/man2/getrlimit.2.html)
- [`epoll(7)`](https://man7.org/linux/man-pages/man7/epoll.7.html)
- [`accept(2)` — error list, including `EMFILE`](https://man7.org/linux/man-pages/man2/accept.2.html)
- [Node `http.Agent`](https://nodejs.org/api/http.html#class-httpagent)
