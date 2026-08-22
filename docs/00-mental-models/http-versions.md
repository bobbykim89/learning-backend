---
title: HTTP/1.1 vs HTTP/2 vs HTTP/3
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

Three wire formats for the same semantics. Methods, status codes, and headers mean
the same thing in all three — that shared meaning is
[HTTP semantics](http-semantics.md), specified separately for exactly this reason.
What changes is how bytes are framed and how many requests can share a connection.

**HTTP/1.1** is a text protocol, one request outstanding per connection at a
time. Pipelining was specified but broken by intermediaries in practice, so
clients treat a connection as serial. Browsers therefore open around six
connections per host, and a generation of frontend optimisation — concatenating
files, spriting images, sharding across domains — existed to work around a limit
of six.

**HTTP/2** keeps the semantics and replaces the framing with a binary protocol.
Many **streams**, each an independent request-response, are multiplexed over one
TCP connection. Headers are compressed with HPACK, which matters because headers
are repetitive and, on a request for a small asset, often larger than the body.
Flow control exists per stream and per connection.

The critical caveat: **HTTP/2 removes head-of-line blocking at the HTTP layer and
not at the TCP layer.** All streams ride one TCP connection, and TCP guarantees
in-order delivery for the whole connection. One lost packet stalls every stream,
including those whose data already arrived — see
[TCP fundamentals](tcp-fundamentals.md).

**HTTP/3** fixes that by abandoning TCP. It runs over **QUIC**, which is built on
UDP and implements reliability per stream, so a loss affecting one stream does not
stall the others. TLS 1.3 is integrated rather than layered beneath, so the
transport and cryptographic handshakes combine. Connections are identified by a
connection ID rather than the IP-and-port four-tuple, which lets a connection
survive a network change — a phone moving from Wi-Fi to cellular keeps its
connection.

### What actually changes for your server

Mostly less than people expect, and in specific places:

- **Your application code probably never sees it.** In typical deployments h2 and
  h3 are terminated at a CDN or load balancer, which speaks HTTP/1.1 to your
  backend. The negotiation, the multiplexing, and the compression all happen in
  front of you.
- **TLS becomes mandatory in practice.** Version selection happens via ALPN during
  the TLS handshake, so h2 and h3 arrive over TLS. Cleartext h2 (`h2c`) exists and
  is rare.
- **Old frontend optimisations become harmful.** Domain sharding multiplies
  connections that h2 was designed to avoid; bundling everything into one file
  defeats fine-grained caching. Under h2 these are anti-patterns, not
  optimisations.
- **Connection counts fall, stream counts rise.** Your file descriptor arithmetic
  improves; your per-connection memory and concurrency limits become the thing to
  watch instead. `SETTINGS_MAX_CONCURRENT_STREAMS` is now a capacity knob.
- **Observability shifts.** A "connection" is no longer a proxy for a request.
  Tooling for h3 is less mature — `tcpdump` on UDP gives you encrypted QUIC, and
  fewer middleboxes can inspect it.
- **CPU cost.** QUIC's transport runs in userspace rather than the kernel, so it
  generally costs more CPU per byte than TCP. That is a real operational number at
  scale.

## Why it exists (what came before)

HTTP/0.9 was one request per connection, then close. HTTP/1.0 added headers and
status codes; HTTP/1.1 added persistent connections, virtual hosting via `Host`,
and chunked transfer encoding.

By the 2010s the binding constraint on page load was not bandwidth but round
trips: dozens of small assets, six connections, each with a handshake, each
serialised. Google's SPDY demonstrated that multiplexing over one connection was
substantially faster, and SPDY became HTTP/2 with relatively few changes.

HTTP/2 then revealed the next layer's limit. Multiplexing many streams on one TCP
connection concentrates the cost of any single packet loss, so on lossy networks
h2 could be *worse* than several HTTP/1.1 connections. Fixing that required
replacing TCP, which was impossible to do in the kernel across the internet — so
QUIC was built on UDP, in userspace, where it could be deployed by shipping a
library. Middlebox ossification, not protocol design, is why HTTP/3 rides UDP.

## Smallest example

Observing which version is in play, and forcing each:

```bash
# What did ALPN negotiate?
curl -sI --http2 https://example.com | head -1        # HTTP/2 200
curl -sI --http3 https://example.com | head -1        # needs curl built with HTTP/3
curl -sI --http1.1 https://example.com | head -1      # HTTP/1.1 200

# The offer of an h3 upgrade is advertised in a response header
curl -sI https://example.com | grep -i '^alt-svc'     # e.g. h3=":443"; ma=86400

# Which protocols does the server offer via ALPN?
openssl s_client -connect example.com:443 -servername example.com -alpn h2,http/1.1 \
  </dev/null 2>/dev/null | grep -i 'ALPN'
```

```javascript
// An HTTP/2 client, to see streams rather than connections
import http2 from "node:http2";

const client = http2.connect("https://example.com");
// Two requests, one connection, concurrent streams:
for (const path of ["/", "/"]) {
  const req = client.request({ ":path": path });
  req.on("response", (h) => console.log(path, h[":status"]));
  req.on("end", () => {});
  req.resume();
  req.end();
}
setTimeout(() => client.close(), 2000);
```

```python
# httpx negotiates h2 via ALPN when the extra is installed
import httpx
with httpx.Client(http2=True) as c:
    r = c.get("https://example.com")
    print(r.http_version)      # "HTTP/2" or "HTTP/1.1"
```

## Tradeoffs & when it's wrong

**HTTP/2 over HTTP/1.1.** Right for many small resources, which is most web
traffic: one handshake, one warmed congestion window, compressed headers. It is
roughly neutral for a single large download, where multiplexing buys nothing. It
is **worse** on high-loss links than several parallel HTTP/1.1 connections,
because loss now stalls every stream instead of one of six.

**HTTP/3 over HTTP/2.** Right on lossy and mobile networks, and where connection
migration matters. It costs more CPU, weaker tooling, and a UDP dependency: some
corporate and public networks block or throttle UDP on 443, so h3 must always be
able to fall back. It is not a meaningful win on a clean, low-loss datacentre path
— which is exactly where backend-to-backend traffic lives.

**Terminating at the edge versus end-to-end h2.** Terminating means your backend
stays simple and your protocol support is someone else's problem. End-to-end h2
between your own services can be worthwhile for gRPC-style workloads, where
multiplexing many concurrent calls over one connection genuinely matters — that
is why gRPC is built on h2.

What would change these answers: loss rate, resource count and size, and whether
clients are mobile. Not general "newer is better" reasoning.

## Failure modes & operational cost

- **HTTP/2 Rapid Reset (CVE-2023-44487).** An attacker opens streams and cancels
  them immediately; cancelled streams free the concurrency slot while the server
  has already begun work, so the effective request rate vastly exceeds
  `MAX_CONCURRENT_STREAMS`. Mitigation is rate-limiting resets, not raising limits.
  A good illustration of the general point: multiplexing makes a single connection
  a much more powerful lever for an attacker.
- **Header compression state as a memory target.** HPACK and QPACK keep per-
  connection dynamic tables; a malicious peer can inflate memory or cause
  expensive decompression.
- **Concurrency limits as a hidden queue.** With streams capped, excess requests
  wait invisibly on the client side. Latency rises with no server-side sign.
- **UDP blocked or throttled.** h3 fails or degrades on some networks, so
  Alt-Svc-driven fallback must work. Test it deliberately.
- **Amplification.** QUIC servers must limit how much they send to an unvalidated
  address, or become a reflection vector. A concern for implementors, and for
  anyone tuning one.
- **Loss of the connection-equals-request assumption.** Rate limits, connection
  caps, and logs built around HTTP/1.1's model quietly stop measuring what they
  used to.
- **Downgrade surprises.** Some intermediaries silently fall back to 1.1, so
  behaviour differs between environments in ways that are invisible until you
  check the negotiated version.

## Open questions / to verify

Not checked against primary sources in this session:

- Whether HTTP/2 server push is fully removed from major browsers and current
  server defaults, or merely disabled by default.
- The default `SETTINGS_MAX_CONCURRENT_STREAMS` in the servers and proxies in use
  here, and whether reset-flood mitigations are enabled by default.
- Whether HTTP/2 priority signalling is implemented anywhere in this path, given
  the original scheme's deprecation in favour of extensible priorities.
- HTTP/3 support status in the specific client libraries in play (`httpx`, Node's
  built-ins, `curl` as packaged here) — h3 support is frequently absent from
  default builds.
- Whether the CDN or load balancer in front of this service terminates h2/h3 and
  what it speaks to the origin, since that determines whether any of this reaches
  application code.
- Relative CPU cost of QUIC versus TCP on this hardware, measured rather than
  assumed.

Candidate for `practiced`: serve the same page over 1.1 and h2 and compare load
time with 50 small assets, then repeat with 3% simulated packet loss (`tc netem`)
and see the ordering reverse. Report both numbers.

## Sources

- [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112)
- [RFC 9113 — HTTP/2](https://www.rfc-editor.org/rfc/rfc9113)
- [RFC 9114 — HTTP/3](https://www.rfc-editor.org/rfc/rfc9114)
- [RFC 9000 — QUIC](https://www.rfc-editor.org/rfc/rfc9000)
- [CVE-2023-44487 — HTTP/2 Rapid Reset](https://nvd.nist.gov/vuln/detail/CVE-2023-44487)
