---
title: HTTP semantics in depth
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

The meanings HTTP assigns to methods, status codes, and headers — independent of
the wire format carrying them, which is [HTTP versions](http-versions.md). These
meanings are not conventions you may reinterpret locally: caches, proxies,
browsers, crawlers, and retry logic all act on them without asking you.

### Safe, idempotent, neither

Two distinct properties, routinely conflated:

- **Safe** — the request is not intended to change state. `GET`, `HEAD`,
  `OPTIONS`, `TRACE`.
- **Idempotent** — issuing it N times has the same effect as issuing it once.
  All safe methods, plus `PUT` and `DELETE`.
- **Neither** — `POST`, `PATCH`.

| Method | Safe | Idempotent | Body | Cacheable |
|---|---|---|---|---|
| GET | yes | yes | no | yes |
| HEAD | yes | yes | no | yes |
| OPTIONS | yes | yes | no | no |
| PUT | no | yes | yes | no |
| DELETE | no | yes | no | no |
| POST | no | **no** | yes | rarely |
| PATCH | no | **no** | yes | no |

Why the distinction has teeth:

**Retries are only safe for idempotent methods.** Every client library, proxy, and
service mesh will retry an idempotent request on a timeout. A timeout does not
tell you whether the server processed the request — only that you did not hear
back. Retrying a `POST` may charge a card twice, which is why
[idempotency keys](../05-api-design/api-idempotency.md) exist as an explicit
mechanism rather than an assumption.

**`DELETE` is idempotent, and that shapes its status code.** Deleting an
already-deleted resource should not be an error; the end state is what idempotency
is about.

**A `GET` with side effects will be triggered by things that are not users.**
Crawlers, link previews in chat apps, browser prefetching, and antivirus scanners
all issue `GET` requests freely. `GET /users/42/delete` is a bug that fires when
someone pastes a link into Slack.

### Status codes worth choosing deliberately

The distinctions that carry real behaviour:

**2xx.** `201 Created` should carry a `Location` header pointing at the new
resource. `202 Accepted` means "not done yet" and belongs with a status resource
to poll. `204 No Content` means success with deliberately no body — not an empty
`200`.

**3xx — and this is where method rewriting lives.** Historically `301` and `302`
caused clients to rewrite a `POST` into a `GET` when following the redirect. `307`
and `308` were introduced precisely to preserve the method and body. So:

| Code | Permanence | Method preserved |
|---|---|---|
| 301 | permanent | no, in practice rewritten to GET |
| 302 | temporary | no, in practice rewritten to GET |
| 307 | temporary | yes |
| 308 | permanent | yes |

Redirecting a form `POST` with `302` silently converts it to a `GET` and drops the
body.

**4xx.** `400` is malformed; `422` is well-formed but semantically invalid — a
useful split when validation errors need structure. `401` means "not
authenticated" and must carry `WWW-Authenticate`; `403` means "authenticated and
still not allowed". Returning `404` instead of `403` deliberately hides existence,
which is enumeration resistance rather than a mistake. `405` must carry `Allow`.
`409` is a conflict with current state; `412` is a failed precondition from a
conditional header; `429` should carry `Retry-After`.

**5xx.** `500` is your bug; `502` is a bad response from upstream; `503` is
unavailable and should carry `Retry-After`; `504` is an upstream timeout. The
distinction matters because it tells the caller whether retrying could possibly
help.

**The anti-pattern:** `200 OK` with `{"error": "not found"}` in the body. Caches
will store it, retry logic will treat it as success, and every dashboard will show
a healthy service.

### Header taxonomy

Four groups worth separating:

- **Representation metadata** — describes the body: `Content-Type`,
  `Content-Encoding`, `Content-Language`, `ETag`, `Last-Modified`.
- **Conditional** — makes a request depend on state: `If-Match`,
  `If-None-Match`, `If-Modified-Since`, `If-Unmodified-Since`. These are the
  mechanism for optimistic concurrency over HTTP: `If-Match` with an ETag gives
  you compare-and-swap, answered with `412` on conflict.
- **Negotiation and caching** — `Accept*`, `Cache-Control`, and `Vary`. `Vary` is
  the one people omit: it tells caches which request headers the response depended
  on. Omitting `Vary: Authorization` on a per-user response is how a shared cache
  serves one user's data to another.
- **Hop-by-hop** — meaningful only for a single connection and **must not** be
  forwarded by a proxy: `Connection`, `Keep-Alive`, `Transfer-Encoding`, `TE`,
  `Upgrade`, `Trailer`, `Proxy-Authenticate`. Everything else is end-to-end.

### Message framing, and how it becomes a vulnerability

This is the part that matters most, and the answer to the question you got stuck
on earlier.

A response or request body must be delimited exactly one of two ways:

- `Content-Length: N` — exactly N bytes follow.
- `Transfer-Encoding: chunked` — a sequence of size-prefixed chunks, terminated by
  a zero-length chunk.

**A message must not carry both.** If it does, the specification says
`Transfer-Encoding` wins and `Content-Length` must be ignored or the message
rejected. That "or" is the problem: real deployments contain more than one HTTP
parser — a CDN, a load balancer, your framework — and if any two of them resolve
the ambiguity differently, they disagree about **where the request ends**.

That disagreement is **HTTP request smuggling**. The variants are named for which
parser trusts which header: `CL.TE`, `TE.CL`, `TE.TE`.

Here is the mechanism, concretely. Suppose a front-end proxy uses
`Content-Length` and the back-end uses `Transfer-Encoding`, and the proxy reuses
one connection to the back-end for many users' requests. An attacker sends a
single request crafted so the proxy forwards all of it, but the back-end considers
the request finished early. The remaining bytes sit in the back-end's buffer.

When the next request arrives on that pooled connection — **someone else's
request** — the back-end prepends the leftover bytes to it. The victim's request
line is now appended to the attacker's partial request. The attacker controls the
method, the path, and the headers that the victim's request is interpreted with;
the victim's own credentials come along. Depending on the shape, this leaks the
victim's request to the attacker or executes the attacker's request with the
victim's session.

Two conditions are load-bearing, and both were present in the scenario earlier:

1. **Disagreement about where a request ends** — which a server doing one `recv()`
   and no framing at all has by construction, since it never determines an end at
   all.
2. **A connection reused across trust boundaries** — keep-alive on a shared,
   pooled connection. `Connection: close` after every request removes the second
   condition, which is why the toy server was not directly exploitable and why
   turning on keep-alive would have made it so.

So the answer to "two pipelined requests in one packet, one `recv`, keep-alive":
the server reads both, answers the first, and the bytes of the second are never
interpreted as a request. The client waits for a second response that never
comes. And if that socket is returned to a shared pool, whatever is still
unconsumed becomes the prefix of the next request on it.

The related bug is **response splitting**: unescaped CR/LF in a value you place
into a response header lets an attacker terminate the headers early and inject an
entire second response, which caches may then store.

## Why it exists (what came before)

HTTP began as one method and no headers. Everything above — status classes, method
properties, conditional requests, content negotiation — accreted, which is why the
current specifications are a *reorganisation* rather than a redesign: RFC 9110
holds semantics for all versions, RFC 9112 the HTTP/1.1 wire format. The earlier
RFC 2616 mixed the two, which is a large part of why implementations disagreed
about framing in the first place.

Method rewriting on `301`/`302` is a fossil: the specification said preserve the
method, browsers rewrote it anyway, and `307`/`308` were added to give the
original intent a code that implementations would actually honour.

## Smallest example

Framing and conditional requests, observed:

```bash
# Chunked framing, as bytes on the wire
curl -sv --raw https://example.com 2>&1 | grep -iE 'transfer-encoding|content-length'

# Conditional GET: 200 first, then 304 with no body
etag=$(curl -sI https://example.com | grep -i '^etag' | tr -d '\r' | cut -d' ' -f2)
curl -sI -H "If-None-Match: $etag" https://example.com | head -1     # 304

# Method preservation across redirects: watch what the second request is
curl -sv -X POST -d 'a=1' -L https://httpbin.org/redirect-to?url=/post\&status_code=302 2>&1 \
  | grep -E '^> (POST|GET)'    # 302 -> the follow-up is a GET
```

Compare-and-swap over HTTP, which is the useful thing conditional headers buy:

```python
import httpx

with httpx.Client(base_url="https://api.example.com") as c:
    r = c.get("/items/42")
    etag = r.headers["etag"]

    # Only apply if nobody else changed it since we read it.
    w = c.put("/items/42", json={"name": "new"}, headers={"If-Match": etag})
    if w.status_code == 412:
        print("lost the race; re-read and retry")
```

```javascript
// Correct framing when writing a response by hand: pick exactly one.
res.writeHead(200, { "content-type": "application/json",
                     "content-length": Buffer.byteLength(body) });
res.end(body);

// Or stream, and let chunked framing do the delimiting — never set both.
res.writeHead(200, { "content-type": "application/json" });
stream.pipe(res);
```

## Tradeoffs & when it's wrong

**Rich status codes versus a small set.** Precise codes let generic
infrastructure behave correctly without knowing your domain — a `429` is
understood by every retry layer. The counter-argument is real: clients frequently
mishandle anything beyond `200`/`400`/`500`, and some misinterpret `422` or `409`
as fatal when a retry would work. Convention beats precision when the primary
client is a single team's mobile app; precision wins when the API is public and
sits behind shared infrastructure.

**`404` versus `403` for a forbidden resource.** `404` resists enumeration and
lies to legitimate clients, which makes debugging authorisation harder. `403` is
honest and confirms existence. The choice depends on whether existence is itself
sensitive.

**`PUT` versus `PATCH`.** `PUT` is idempotent and therefore safely retryable, at
the cost of requiring the whole representation and risking clobbering concurrent
edits. `PATCH` sends less and is not idempotent, so it needs an idempotency key or
a precondition to be retry-safe.

What would change these answers: who the clients are, and whether anything between
you and them acts on the semantics.

## Failure modes & operational cost

- **Request smuggling** from framing disagreement across parsers. Cross-user
  impact, and reachable through any intermediary.
- **Response splitting** from unescaped CR/LF in header values, poisoning caches.
- **Cache poisoning from a missing `Vary`.** One user's authenticated response
  served to another. Silent, and catastrophic.
- **`200` with an error body.** Breaks caches, retries, alerting, and SLOs at once.
- **Non-idempotent operations behind an idempotent method.** A `GET` that mutates
  gets fired by prefetchers and crawlers.
- **Retrying `POST` without an idempotency key.** Duplicate charges, duplicate
  orders. The retry is usually in a library you did not configure.
- **Method rewritten by a `302`.** Form submissions silently become `GET`s and
  bodies vanish.
- **Both `Content-Length` and `Transfer-Encoding` accepted.** Reject such requests
  outright; do not attempt to reconcile them.
- **Trusting `X-Forwarded-For` unconditionally.** Client-supplied, therefore
  spoofable unless the edge overwrites it.
- **Header size limits.** Defaults are small enough that large cookies or JWTs
  produce `431` or `502` from an intermediary rather than a clear application error.

## Open questions / to verify

Not checked against primary sources in this session:

- Whether the frameworks in use here reject a request carrying both
  `Content-Length` and `Transfer-Encoding` by default, or silently prefer one.
- Whether current browsers still rewrite `POST` to `GET` on `301`/`302` in all
  cases, or whether behaviour has converged on preserving the method.
- Default maximum header size in the servers and proxies in this path, and which
  status is returned when it is exceeded.
- Whether `PATCH` is registered as cacheable or conditional-capable in RFC 9110's
  method registry, and what `PATCH` with `If-Match` is specified to do.
- Whether any intermediary in this deployment normalises or strips hop-by-hop
  headers as required.
- Exactly which status codes the retry logic in the HTTP clients used here treats
  as retryable, by default.

Candidate for `practiced`: build the two-parser desync deliberately in a local
lab — a proxy preferring `Content-Length` in front of a server preferring
`Transfer-Encoding`, with a pooled upstream connection — and demonstrate one
client's request being prefixed by another's bytes. Then fix it by rejecting
ambiguous framing, and confirm the attack stops working.

## Sources

- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9112 — HTTP/1.1, message framing in §6](https://www.rfc-editor.org/rfc/rfc9112#section-6)
- [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [RFC 7231 §6 — status code registry history](https://www.rfc-editor.org/rfc/rfc7231)
