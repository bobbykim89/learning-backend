---
title: TLS fundamentals
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

A layer between TCP and your application protocol that provides three things:
**confidentiality** (an observer cannot read the traffic), **integrity** (an
observer cannot alter it undetected), and **authentication** of the server's
identity — and optionally the client's.

The third is the one people under-appreciate. Encryption without authentication
is nearly worthless: an attacker who can intercept the connection simply presents
their own key and encrypts to themselves. The certificate chain is what makes the
encryption mean something.

### The handshake

**TLS 1.3, one round trip:**

1. **ClientHello** — supported versions, cipher suites, a key share (the client
   guesses which group the server will pick), plus two extensions worth knowing by
   name: **SNI**, the hostname being requested, and **ALPN**, the application
   protocol being negotiated (`h2`, `http/1.1`).
2. **ServerHello** — chosen parameters, the server's key share, then the
   certificate chain and a signature, already encrypted.
3. **Client Finished** — and application data may flow.

**TLS 1.2 takes two round trips**, because the client cannot guess the key
exchange parameters and must wait for the server's choice before sending its
share.

Two consequences fall out of the details:

**SNI is sent in the clear.** It has to be — the server needs to know which
certificate to present before it has established a secure channel. So the
hostname you are visiting is visible to the network even though the traffic is
not. Encrypted Client Hello exists to close this, and is not yet universal.

**ALPN happens here, not in HTTP.** Whether you speak HTTP/2 is decided during
the TLS handshake. That is why there is no practical HTTP/2 upgrade dance over
TLS, and why h2 without TLS (`h2c`) is rare in the wild.

**Forward secrecy.** Modern handshakes use ephemeral Diffie-Hellman (ECDHE): the
session key is derived from keys thrown away afterwards. Recording the traffic
and later stealing the server's private key does not decrypt it. TLS 1.3 removed
static RSA key transport entirely, making this mandatory rather than a
configuration choice.

**The certificate signs; it does not encrypt.** The private key's job in a modern
handshake is to sign a transcript, proving the server possesses it. It is not used
to encrypt the session key.

**0-RTT** in TLS 1.3 lets a resuming client send application data with its first
flight, saving the round trip. The catch is real: 0-RTT data can be replayed by an
attacker, so it is only safe for genuinely idempotent requests. Sending a POST in
0-RTT is a correctness bug waiting for a replay.

### Certificates and the chain of trust

A certificate binds an identity to a public key, signed by someone else. Chains
run leaf → intermediate(s) → root, where the root is in the client's trust store
and the intermediates are not.

**The server must send the intermediates.** This produces one of the most
instructive misconfigurations in the field: browsers can often fetch a missing
intermediate on their own (via the certificate's AIA extension) and will render
the site fine, while `curl`, Java clients, Go clients, and mobile SDKs fail
outright. "It works in my browser but the API client gets a TLS error" is
almost always a missing intermediate.

What a client validates:

- **Signature chain** up to a trusted root.
- **Hostname**, against the Subject Alternative Name extension. The legacy Common
  Name field is ignored by modern clients — a certificate with the right CN and no
  matching SAN fails.
- **Validity dates**, which is why **clock skew breaks TLS**. A machine whose
  clock is wrong by days cannot connect to anything, and the error blames the
  certificate.
- **Revocation**, the weakest link: CRLs are large, OCSP requires a live lookup,
  OCSP stapling has the server fetch and attach the proof itself. The industry has
  been drifting toward short certificate lifetimes instead, on the grounds that
  expiry is a revocation mechanism that actually works.

### Termination points

Where TLS ends is a deployment decision with security consequences:

- **At the edge / load balancer.** Certificates live in one place, and the
  balancer can read headers to route. The internal hop is plaintext, and a second
  HTTP parser now sits in the path — which must agree with yours about message
  boundaries, or you have a [request smuggling](http-semantics.md) exposure.
- **Re-encrypted to the backend.** Edge terminates, then opens a new TLS
  connection inward. Encrypted throughout, at the cost of two handshakes and a
  second certificate story.
- **Passthrough.** The balancer forwards TCP without decrypting. End-to-end, but
  the balancer cannot route on anything above TCP, and the application owns
  certificates.
- **Sidecar / service mesh.** A local proxy terminates and originates, so
  application code speaks plaintext to localhost while the network sees mTLS. The
  cost is the mesh itself.

**mTLS** adds client authentication: the server requests a certificate and
validates it against a CA it trusts. This is how service-to-service
authentication works without shared secrets. The operational cost is entirely
about lifecycle — issuing, distributing, and rotating certificates for every
workload, and the outage when rotation is missed.

## Why it exists (what came before)

HTTP was plaintext, and remained so as it started carrying credentials and card
numbers. SSL was bolted on beside it rather than into it, which is why TLS sits as
a separate layer between TCP and HTTP and why "HTTPS" is just HTTP over that
layer.

The version history is a record of breaks. SSL 2.0 and 3.0 are broken (POODLE),
TLS 1.0 and 1.1 are deprecated, and TLS 1.2's flexibility was itself a liability —
too many negotiable options, several of them unsafe. TLS 1.3's design response was
subtraction: fewer cipher suites, no static RSA key exchange, no renegotiation, no
compression, mandatory forward secrecy. That is why it is both faster and simpler
to configure safely.

## Smallest example

Inspecting a real handshake is more instructive than any diagram:

```bash
# Full handshake and chain. -servername sets SNI, which you must do explicitly.
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null

# What the negotiated result was
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | grep -E 'Protocol|Cipher|Verify return code'

# Certificate details that actually get validated
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# Is the chain complete as served? (fails if an intermediate is missing)
openssl s_client -connect example.com:443 -servername example.com -verify_return_error </dev/null
```

```python
import socket, ssl

ctx = ssl.create_default_context()      # verifies chain AND hostname by default
with socket.create_connection(("example.com", 443), timeout=5) as raw:
    with ctx.wrap_socket(raw, server_hostname="example.com") as tls:
        print(tls.version(), tls.cipher()[0])
        print(tls.getpeercert()["subjectAltName"][:3])

# The dangerous version, for contrast — never in production:
bad = ssl.create_default_context()
bad.check_hostname = False
bad.verify_mode = ssl.CERT_NONE         # encrypted, unauthenticated, worthless
```

```javascript
import { connect } from "node:tls";

const sock = connect({ host: "example.com", port: 443, servername: "example.com",
                       ALPNProtocols: ["h2", "http/1.1"] }, () => {
  console.log(sock.authorized, sock.getProtocol(), sock.alpnProtocol);
  console.log(sock.getPeerCertificate().subjectaltname);
  sock.end();
});
```

Note that `server_hostname` / `servername` is required, not optional: it sets SNI
*and* it is what hostname verification checks against. Omitting it on a
multi-tenant host gets you the wrong certificate and a confusing failure.

## Tradeoffs & when it's wrong

**Terminating at the edge** centralises certificates and enables routing; it
concedes a plaintext internal hop and adds a parser that must agree with yours.
Right in most deployments where the internal network is trusted-ish and
operational simplicity matters.

**mTLS everywhere** removes shared secrets and gives strong workload identity. It
costs a certificate lifecycle for every service, and its failure mode is a total
outage on expiry rather than a degraded one. What would change the answer: whether
you have automated issuance and rotation. Without that, mTLS is an outage
scheduled for a date you have forgotten.

**Certificate pinning** defeats a compromised CA. It also breaks your service when
you rotate, and the broken clients are the ones you cannot update — mobile apps in
the field. Rarely worth it outside high-threat contexts.

**0-RTT** saves a round trip and admits replay. Right for idempotent GETs, wrong
for anything that mutates.

**Short-lived certificates** make revocation mostly unnecessary and force
automation, which is a benefit disguised as a cost. They are wrong wherever
issuance cannot be automated, because then you have merely shortened the fuse.

## Failure modes & operational cost

- **Expiry.** Still the most common cause of TLS outages, and it takes down
  everything at once, at a predictable moment nobody was watching. Alert on days
  remaining, not on failure.
- **Missing intermediate.** Works in browsers, fails in API clients. Test with
  `openssl s_client -verify_return_error`, not with a browser.
- **Hostname / SAN mismatch.** Especially after adding a domain and reissuing
  without all names.
- **Clock skew.** Certificate validity cannot be checked correctly by a machine
  with the wrong time; symptoms blame the certificate.
- **Missing SNI on a multi-tenant host.** The server presents a default
  certificate, and validation fails against a name it does not cover.
- **Protocol or cipher mismatch.** Hardening to TLS 1.3-only silently excludes old
  clients, embedded devices, and some corporate middleboxes.
- **OCSP responder latency.** With soft-fail clients this is a hang; with hard-fail
  it is an outage caused by someone else's infrastructure. Stapling moves that
  dependency to your server.
- **Handshake CPU cost.** Real at high connection rates, which is another reason
  connection reuse matters — see [TCP fundamentals](tcp-fundamentals.md).
- **`verify_mode = CERT_NONE` in test code reaching production.** Encryption with
  no authentication, which looks identical in a green test suite.

## Open questions / to verify

Not checked against primary sources in this session:

- Minimum TLS version Python's `ssl.create_default_context()` permits on 3.13, and
  Node's default `minVersion`.
- Whether OCSP stapling is enabled on the infrastructure used here, and whether
  the clients in play are soft-fail or hard-fail.
- Current browser and CA policy on maximum certificate lifetime — the direction of
  travel is shorter, but the specific number needs checking before it is quoted.
- Whether Encrypted Client Hello is available in the TLS stacks used here, and
  what fraction of the path supports it.
- Whether Python's `ssl` module verifies hostname before or after chain
  validation, which affects which error you see first when both are wrong.
- Whether `getpeercert()` returns the full chain as served or only the leaf.

Candidate for `practiced`: stand up a server with a deliberately incomplete chain,
confirm it succeeds in a browser and fails with `curl`, then fix it and verify with
`openssl s_client -verify_return_error`. Also set the clock forward past expiry and
observe which error the client reports.

## Sources

- [RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [RFC 6125 — Identity verification (SAN vs CN)](https://www.rfc-editor.org/rfc/rfc6125)
- [RFC 7301 — ALPN](https://www.rfc-editor.org/rfc/rfc7301)
- [Python `ssl`](https://docs.python.org/3/library/ssl.html)
- [Node `tls`](https://nodejs.org/api/tls.html)
