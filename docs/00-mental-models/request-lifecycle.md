---
title: The lifecycle of a request
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

The path a request takes from a URL to a response, and back. The application
handler is a small step in the middle; most latency and most failure modes live
outside it.

**Client side.** Parse the URL → resolve the hostname → open a TCP connection →
TLS handshake → write request bytes.

Everything here happens before any application code exists:

- **DNS** is a network call with multi-layer caching and TTLs, and may return
  several addresses. The POSIX resolver call is synchronous, so runtimes hide it
  in a thread pool rather than doing true async I/O.
- **TCP connect** costs one round trip: SYN, SYN-ACK, ACK.
- **TLS** costs one more round trip on 1.3, two on 1.2. The `ClientHello`
  carries SNI (which hostname, so one IP can serve many certificates) and ALPN
  (which negotiates HTTP/1.1 vs h2 at this point, not later).

A cold HTTPS connection therefore spends roughly 2–3 round trips before the
handler exists. A warm keep-alive connection spends none. That gap is why
connection reuse is one of the highest-leverage performance levers in backend
work, and it is invisible from inside a handler.

**Server side.**

1. The process created a **listening socket**, bound it to a port, and called
   `listen(backlog)`.
2. On connect, **the kernel completes the TCP handshake itself** and parks the
   finished connection in an **accept queue**. The application is not involved.
   If the application is slow or stuck, connections keep completing and piling
   up here until the queue fills, after which new connections are dropped.
3. The process calls `accept()` and receives **a new file descriptor** for that
   one connection. The listening socket is unchanged.
4. It reads bytes off that descriptor. Those bytes are a text protocol; parsing
   them into a request object is what a framework does.
5. The handler runs, as one worker, thread, or coroutine among N. Whether it
   blocks the other N−1 is the subject of [process vs thread vs
   coroutine](process-thread-coroutine.md), then [Python's](python-execution-model.md)
   and [Node's](node-execution-model.md) execution models.
6. It writes response bytes back. **Framing** tells the client when the body
   ends: `Content-Length` or `Transfer-Encoding: chunked`. Get it wrong and the
   client waits for bytes that never come.
7. The connection is reused (HTTP/1.1 keep-alive is the default) or closed.
   Whichever side closes actively holds the socket in `TIME_WAIT` afterward.

The load-bearing idea: **a connection is a file descriptor**, which is why
[running out of them](sockets-and-file-descriptors.md) is a real failure mode.

## Why it exists (what came before)

Each layer exists because the one below it does not provide something:

- **IP** routes packets and may drop, duplicate, or reorder them.
- **TCP** exists to turn that into an ordered, gap-free byte stream. It
  deliberately does not add message boundaries — that would presume a
  message shape it cannot know.
- **DNS** exists to decouple names from addresses, so a service can move,
  scale, or fail over without clients changing.
- **TLS** was added around an existing plaintext protocol, which is why it sits
  between TCP and HTTP and why termination point is a deployment decision
  rather than an application one.
- **HTTP** supplies the framing TCP omits, plus semantics: methods, status
  codes, headers.

Before keep-alive (HTTP/1.0 and earlier), every object cost a fresh TCP
connection and handshake, which is why pages were built to minimise request
count. Before worker models, CGI forked a process per request — correct, and
expensive enough to shape a decade of architecture, and the direct ancestor of
[the pre-fork model](wsgi-asgi-and-clustering.md) still in use today.

## Smallest example

The same server twice, no framework and no dependencies. This is what a
framework wraps.

```python
import socket

srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(("127.0.0.1", 8080))
srv.listen(128)                        # accept-queue depth

while True:
    conn, addr = srv.accept()          # blocks until the kernel hands us one
    data = conn.recv(65535)            # BUG: one read is not one request
    body = b"hi\n"
    conn.sendall(
        b"HTTP/1.1 200 OK\r\n"
        b"Content-Length: " + str(len(body)).encode() + b"\r\n"
        b"Connection: close\r\n"
        b"\r\n" + body
    )
    conn.close()
```

```javascript
import { createServer } from "node:net";

const srv = createServer((conn) => {   // conn is a socket, not a "request"
  conn.once("data", (data) => {        // BUG: same assumption as above
    const body = "hi\n";
    conn.end(
      "HTTP/1.1 200 OK\r\n" +
      `Content-Length: ${Buffer.byteLength(body)}\r\n` +
      "Connection: close\r\n" +
      "\r\n" + body
    );
  });
});

srv.listen(8080, "127.0.0.1", 128);
```

`SO_REUSEADDR` lets the port be rebound while old connections linger in
`TIME_WAIT`, which is why a restart does not fail with "address already in use".

## Tradeoffs & when it's wrong

**Hand-rolling the socket layer** is right for learning it, for implementing a
non-HTTP protocol, and where a framework denies needed control. It is wrong for
anything user-facing: framing, chunked encoding, header limits, and timeouts are
security-relevant and tedious to get right.

What would change that: a protocol that is simple and entirely yours — a
length-prefixed internal wire format — removes the boundary ambiguity that makes
hand-rolled HTTP dangerous.

**Keep-alive vs close.** Reuse removes 2–3 round trips per request and is
almost always right. It costs a held file descriptor per idle connection, and it
is the precondition for request-smuggling bugs when framing is wrong.

**TLS termination point.** Terminating at a proxy centralises certificates and
lets the proxy inspect and route; it means plaintext on the internal hop, and it
puts a second HTTP parser in the path that must agree with yours about where
requests end. Terminating in the application keeps the path encrypted end to end
at the cost of certificate distribution.

## Failure modes & operational cost

Three failures that look similar and are not:

| Symptom | Meaning | Who answered |
|---|---|---|
| `Connection refused` | Nothing bound to that port; kernel replies RST | Kernel, no process involved |
| Connects, then hangs | Something accepted it and never responded, or a firewall dropped it | The application, or nobody |
| `502` / `503` | A proxy connected fine and its own upstream failed | The proxy |

**One read is not one request.** Verified against the server above: sending
`GET / HTTP/1.1\r\nHost: x\r\n` with no terminating blank line still returns
`200 OK`, and a `POST` with `Content-Length: 11` returns `200 OK` without the
body ever being read. TCP guarantees order and completeness, never boundaries.
Correct handling reads until `\r\n\r\n`, then reads exactly `Content-Length`
bytes, or handles `chunked`.

When two parties disagree about where a request ends and the connection is
reused, the leftover bytes become the start of the next request — **HTTP request
smuggling**, where an attacker controls the prefix of a request that may belong
to another user. `Connection: close` blunts the exploit; the defect stays.

**No read timeout on a serial accept loop.** Verified: one connection that
sends nothing blocks every other client, because the process sits in `recv()`
and never returns to `accept()`. Those clients are not refused — the kernel
completes their handshakes and queues them — they simply get no response. This
is slowloris, and it needs timeouts plus concurrency, not a bigger backlog.

**Accept-queue overflow.** The backlog bounds how many completed connections
may wait. Full queue means new connections are dropped, and clients see a hang
or a refusal depending on the platform. A larger backlog buys time; it does not
fix a handler that is too slow.

**Ephemeral port exhaustion.** Each connection is a four-tuple, and the client
side draws its port from the ephemeral range. `Connection: close` at volume
strands ports in `TIME_WAIT` on whichever side closed, which is a real limit on
busy proxies and load generators.

## Open questions / to verify

Everything here is version- or platform-specific and was not checked against
primary docs in this session:

- Default `net.core.somaxconn` on this kernel, and how it caps the `listen()`
  backlog argument.
- `TIME_WAIT` duration on Linux, and whether it is tunable independently of
  `tcp_fin_timeout`.
- Whether Node's `dns.lookup` still uses the libuv thread pool while
  `dns.resolve*` uses c-ares, and the current default pool size.
- Exactly when `nc` half-closes its write side on stdin EOF — it affects
  whether the blocked-server demonstration above blocks in `recv()` or is
  released by EOF. The blocking was observed; the precise mechanism was not
  confirmed.
- Whether `curl` reuses connections across invocations (it does not) versus
  within one invocation with multiple URLs (it does) — worth confirming when
  measuring handshake cost.

Not yet built anything with this, so `status` stays `learning`. Candidate for
`practiced`: extend the server above to frame requests correctly (read to
`\r\n\r\n`, honour `Content-Length`), add a read timeout, and confirm the three
failures above stop reproducing.

## Sources

- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112), message
  framing in §6
- [RFC 9293 — TCP](https://www.rfc-editor.org/rfc/rfc9293)
- [RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [Python `socket` documentation](https://docs.python.org/3/library/socket.html)
- [Node `net` documentation](https://nodejs.org/api/net.html)
