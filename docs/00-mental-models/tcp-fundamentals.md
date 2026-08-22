---
title: TCP fundamentals
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

A protocol that turns IP's unreliable, unordered packet delivery into an ordered,
gap-free byte stream between two endpoints, identified by a four-tuple: source
IP, source port, destination IP, destination port.

Four mechanisms do the work, and each one shows up later as a performance or
failure characteristic:

**Handshake.** SYN, SYN-ACK, ACK — one full round trip before any application
byte may be sent. The kernel maintains two queues during this: a **SYN queue** of
half-open connections mid-handshake, and an **accept queue** of completed
connections waiting for the application to call `accept()`, as described in
[the lifecycle of a request](request-lifecycle.md).

**Reliability.** Every byte has a sequence number; the receiver acknowledges
cumulatively. Loss is detected either by duplicate ACKs (fast retransmit, three
duplicates) or by a retransmission timeout, which backs off exponentially. The
second path is the expensive one: a tail loss with no following packets to trigger
duplicate ACKs waits for the timer, which is why single-packet loss at the end of
a response can cost hundreds of milliseconds.

**Flow control.** The receiver advertises a window — how much it is prepared to
buffer. This protects a slow receiver from a fast sender. The window field is 16
bits, so anything above 64 KB needs the window-scaling option, negotiated at
handshake.

**Congestion control.** Independent of flow control and often confused with it.
Flow control protects the *receiver*; congestion control protects the *network*.
The sender maintains a congestion window, grows it exponentially during slow
start, then linearly, and shrinks it on loss.

Slow start is why a fresh connection is slow for its first few round trips
regardless of available bandwidth — the sender does not yet know what the network
can take. Reusing a warmed connection inherits its congestion window, which is a
second, less obvious reason connection reuse matters.

**Bandwidth-delay product** (bandwidth x RTT) is the amount of data in flight
needed to saturate a path. On a high-bandwidth, high-latency link, a small window
caps throughput no matter how much capacity exists.

### The two "keep-alives" are different things

This confusion causes real misconfiguration, so it is worth stating flatly:

| | What it is | Where configured | Typical default |
|---|---|---|---|
| **TCP keepalive** | Kernel probes an idle connection to see if the peer still exists | `SO_KEEPALIVE` + kernel sysctls | probes begin after ~2 hours |
| **HTTP keep-alive** | Reusing one connection for several HTTP requests | Application / HTTP client | on by default in HTTP/1.1 |

The gap between "~2 hours" and reality is the problem. NAT devices, cloud load
balancers, and firewalls drop idle connections after something like 60–350
seconds, and often drop them **silently** — no FIN, no RST. Both endpoints still
believe the connection is fine. The next write appears to succeed, because it
only reaches the local buffer, and the failure surfaces as a timeout much later.

The fix is application-level heartbeats, or a TCP keepalive interval set well
below the shortest idle timeout in the path. Not the two-hour default.

### Head-of-line blocking

TCP guarantees in-order delivery to the application. If segment 5 is lost,
segments 6 through 20 may have arrived and be sitting in the kernel's buffer, but
your application cannot read them until 5 is retransmitted.

This matters far beyond a single download: HTTP/2 multiplexes many independent
streams onto one TCP connection, and TCP's ordering guarantee applies to the
whole connection. One lost packet stalls **every** stream, including ones whose
data already arrived. Fixing that is the central motivation for QUIC, in
[HTTP versions](http-versions.md).

## Why it exists (what came before)

IP delivers packets and promises nothing: they may be dropped, duplicated,
reordered, or corrupted. Applications that needed a reliable stream all had to
solve the same problems, so the reliability layer was factored out — originally
TCP and IP were one protocol, split precisely so that applications needing
unreliable datagrams could use UDP instead.

Congestion control was not in the original design. It was added after the 1986
NSFNET congestion collapse, when the network's throughput fell by three orders of
magnitude because senders retransmitted into an already-saturated network. That
history is why TCP treats loss as a congestion signal by default — an assumption
that is wrong on wireless links, where loss is often interference, and which
motivates alternatives like BBR that model the path instead.

## Smallest example

The costs, measured rather than described:

```bash
# Where the time actually goes, per phase
curl -w 'dns=%{time_namelookup}s connect=%{time_connect}s tls=%{time_appconnect}s ttfb=%{time_starttransfer}s total=%{time_total}s\n' \
     -o /dev/null -s https://example.com

# Handshake cost, isolated: run twice, second reuses nothing (new process)
# then compare against two URLs in one invocation, which reuses the connection
curl -s -o /dev/null -w 'first=%{time_total}\n' https://example.com
curl -s -o /dev/null -w 'reused=%{time_total}\n' https://example.com https://example.com
```

Disabling Nagle's algorithm, which is the setting most often needed and least
often understood:

```python
import socket

s = socket.create_connection(("example.com", 80))
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)   # send small writes now
s.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
# Linux-specific tuning, well below any NAT idle timeout:
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 30)   # idle before first probe
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)  # between probes
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 3)     # failures before giving up
```

```javascript
import { connect } from "node:net";

const sock = connect(80, "example.com");
sock.setNoDelay(true);          // disable Nagle
sock.setKeepAlive(true, 30_000); // probe after 30s idle
```

**Why `TCP_NODELAY` exists.** Nagle's algorithm coalesces small writes: it holds a
small segment until the previous one is acknowledged, to avoid flooding the
network with tiny packets. Delayed ACK, on the receiver, waits briefly before
acknowledging, hoping to piggyback the ACK on outgoing data. Put them together
and you get a stall: the sender waits for an ACK that the receiver is deliberately
delaying. Classic symptom — a request-response protocol that writes a header and
then a body as two separate writes, showing a consistent ~40 ms pause that
disappears the moment you combine the writes or set `TCP_NODELAY`.

## Tradeoffs & when it's wrong

**TCP versus UDP.** TCP is right whenever you need every byte in order, which is
most application traffic. It is wrong when timeliness beats completeness — live
audio and video, where a retransmitted frame arrives too late to matter, and where
head-of-line blocking is worse than a dropped sample. It is also wrong when you
want to build your own reliability with different tradeoffs, which is exactly what
QUIC does over UDP.

**Nagle on or off.** On saves bandwidth for chatty protocols and is the right
default for bulk transfer. Off is right for interactive request-response, at the
cost of more, smaller packets. What decides it: whether your writes are small and
latency-sensitive.

**Connection reuse.** Nearly always right — it skips the handshake and inherits a
warmed congestion window. It costs a held file descriptor per idle connection
(see [sockets and file descriptors](sockets-and-file-descriptors.md)) and requires
handling connections that died silently.

**Aggressive keepalive probes.** Detect dead peers quickly, at the cost of traffic
on every idle connection — which at a hundred thousand idle connections is not
nothing.

## Failure modes & operational cost

- **Accept queue overflow.** The application is too slow to `accept()`; completed
  connections pile up and then get dropped. Clients see hangs or refusals. A
  larger backlog buys time, not a fix.
- **SYN flood.** The SYN queue fills with half-open connections from spoofed
  sources. SYN cookies are the standard mitigation.
- **Ephemeral port exhaustion.** The client side of every connection draws a port
  from a finite range, and closed connections hold their tuple in `TIME_WAIT`. At
  high connection-per-second rates without reuse, the client runs out. Symptom:
  outbound connections fail on a machine with idle CPU and memory.
- **Silently dead connections.** A middlebox dropped the connection; both ends
  think it is alive. Writes appear to succeed. This is the single most common
  cause of mysterious multi-minute hangs in pooled connections.
- **Retransmission timeouts as latency spikes.** A p99 that is a multiple of the
  p50 with no CPU or database explanation is often loss plus RTO.
- **Buffer bloat.** Oversized buffers along the path mean congestion shows up as
  latency rather than loss, so TCP's loss signal arrives late and queues stay full.
- **Half-open after a crash.** A peer that vanished without a FIN leaves the other
  side holding a connection forever, absent keepalive or an application timeout.

The operational summary: TCP hides enormous complexity successfully, and the price
is that its failures present as *latency* and *hangs* rather than as errors. That
is why timeouts at every layer are non-negotiable.

## Open questions / to verify

Not checked against primary sources in this session:

- Default congestion control algorithm on this kernel (`net.ipv4.tcp_congestion_control`),
  and whether BBR is available.
- Default `net.ipv4.tcp_keepalive_time` / `_intvl` / `_probes`, to confirm the
  ~2-hour figure quoted above.
- Default `net.core.somaxconn` and `tcp_max_syn_backlog`, and how `listen()`'s
  argument interacts with them.
- Default ephemeral port range (`net.ipv4.ip_local_port_range`) and `TIME_WAIT`
  duration here.
- Whether the ~40 ms Nagle/delayed-ACK stall figure matches this platform's
  delayed-ACK timer, or whether that number is Linux-specific folklore.
- Whether `TCP_KEEPIDLE` and friends are available on macOS under those names —
  believed not, which matters for local development parity.

Candidate for `practiced`: reproduce the Nagle/delayed-ACK stall deliberately —
write a header and body as two separate `send()` calls over a real network path,
measure the pause, then fix it two ways (combine the writes, and set
`TCP_NODELAY`) and report both numbers.

## Sources

- [RFC 9293 — TCP](https://www.rfc-editor.org/rfc/rfc9293)
- [RFC 5681 — TCP Congestion Control](https://www.rfc-editor.org/rfc/rfc5681)
- [RFC 896 — Congestion control in IP/TCP internetworks (Nagle)](https://www.rfc-editor.org/rfc/rfc896)
- [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html)
- [Van Jacobson, Congestion Avoidance and Control (1988)](https://ee.lbl.gov/papers/congavoid.pdf)
