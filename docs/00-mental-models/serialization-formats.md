---
title: Serialization formats
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

Turning in-memory values into bytes for transmission or storage, and back. The
choice looks like a performance question and is mostly a **schema evolution**
question: who has to agree with whom about the shape of the data, and what happens
when one side changes first.

Four formats worth knowing, sorted by that axis rather than by speed:

**JSON.** Text, self-describing, no schema. Both sides need no prior agreement,
which is why it won. Its type system is the problem: no integers (every number is
a double), no binary, no date, no decimal.

**MessagePack / CBOR.** Binary, still self-describing, still schemaless. Smaller
and faster than JSON while keeping "you can decode it without knowing anything".
CBOR has an RFC and is used in COSE and WebAuthn; MessagePack is common in caches
and RPC.

**Protobuf.** Schema-first. You write a `.proto`, generate code, and the wire
format carries **field numbers, not field names**. Very compact — varint encoding,
no keys on the wire — and *not* self-describing: bytes without the schema are
close to meaningless.

**Avro.** Also schema-first, but the schema travels with the data — embedded in
object-container files, or referenced through a schema registry, which is how it is
used with Kafka. Its distinguishing feature is explicit **reader/writer schema
resolution**: decoding is defined as reconciling the schema the data was written
with against the one the reader expects.

### The number problem in JSON

Worth its own heading because it bites hard and silently.

JSON has one numeric type, and the JSON specification does not constrain
precision. JavaScript's `number` is an IEEE-754 double, which represents integers
exactly only up to 2^53 − 1. So a 64-bit database ID serialised as a JSON number
and parsed by JavaScript **silently loses precision** above that threshold:

```javascript
JSON.parse('{"id":9007199254740993}').id   // 9007199254740992 — off by one
```

No error, no warning, a wrong ID. The fix is to serialise 64-bit integers as
strings, which every large API eventually does. Money has the same shape of
problem for a different reason: `0.1 + 0.2 !== 0.3` in binary floating point, so
currency belongs in minor units as an integer, or as a string decimal — never a
JSON number.

Binary is the other gap: JSON has no byte type, so binary goes in as base64,
costing about 33% size inflation plus encode and decode work.

### What schema evolution actually requires

The rules differ per format, and they are the practical reason to pick one:

| | Identity of a field | Add a field | Rename a field | Remove a field |
|---|---|---|---|---|
| JSON | its **name** | safe if readers ignore unknowns | breaking | breaking for readers that require it |
| Protobuf | its **number** | safe (new number) | safe — number is identity | reserve the number, never reuse |
| Avro | name, with aliases and defaults | safe with a default | alias | safe if the reader has a default |

The protobuf rule that matters most: **never reuse a field number.** An old client
decoding a new message will interpret number 7 as whatever number 7 used to mean,
with the new type's bytes. That is not an error, it is silent corruption — which is
why `.proto` files carry `reserved` declarations for retired numbers.

The JSON rule that matters most: **readers must ignore unknown fields.** If your
parser is strict, every producer change is a breaking change.

## Why it exists (what came before)

Early RPC used formats generated from interface definitions — CORBA, XML-RPC, SOAP
— which were rigorous, verbose, and required tooling on both ends. XML with schemas
could express nearly anything, at the cost of complexity nobody enjoyed.

JSON's success was ergonomic rather than technical: it was already JavaScript's
object literal, it needed no tooling, and it was readable in a terminal. That
readability is a genuine operational feature, not a weakness — being able to `curl`
an endpoint and see the answer shortens every debugging session.

The binary schema-first formats came back when the scale changed. Once data is
moving between many services and being retained for years in a log, the questions
become "how many bytes" and "can a consumer written last year still read this",
and JSON answers neither well.

## Smallest example

The same record in JSON and protobuf, with the size difference:

```protobuf
// user.proto — the numbers ARE the contract
syntax = "proto3";

message User {
  int64  id    = 1;
  string email = 2;
  bool   active = 3;
  reserved 4;                 // was 'nickname', removed — never reuse 4
}
```

```python
import json

record = {"id": 9007199254740993, "email": "a@example.com", "active": True}

as_json = json.dumps(record).encode()
print(len(as_json), as_json)
# The int survives in Python (arbitrary precision ints), and is destroyed
# by any JavaScript consumer of the same bytes.

# The portable form:
safe = {**record, "id": str(record["id"])}
print(json.dumps(safe))

# Measure rather than assume, on your own shapes:
import timeit
print(timeit.timeit(lambda: json.dumps(record), number=100_000))
```

```javascript
// Where the precision goes
const wire = '{"id":9007199254740993,"email":"a@example.com"}';
console.log(JSON.parse(wire).id);          // 9007199254740992

// Correct handling of 64-bit ids on the wire
console.log(BigInt(JSON.parse('{"id":"9007199254740993"}').id));  // exact

// Dates have no JSON type either — they are strings by convention
console.log(JSON.stringify({ at: new Date() }));  // ISO 8601 string
console.log(typeof JSON.parse('{"at":"2026-08-17T00:00:00Z"}').at); // "string"
```

## Tradeoffs & when it's wrong

**JSON** is right for public APIs, for anything a human debugs, and for
low-to-moderate volume. Its ubiquity means no client needs your tooling — a real
architectural advantage that pure byte-counting misses. It is wrong for
high-volume internal traffic, for long-retained data, and anywhere numeric
precision matters without discipline.

**MessagePack / CBOR** are right when you want JSON's schemalessness with fewer
bytes — a cache payload, an internal RPC where both ends ship together. They are
wrong as a public API format, because you have imposed a decoder on every client
for a modest saving, and wrong where the debuggability of text matters.

**Protobuf** is right for internal service-to-service traffic at volume, and where
a generated client is a benefit rather than a burden. It is wrong when consumers
are outside your control, because they cannot read the bytes without your schema,
and its cost is real: a build step, generated code in your tree, and an evolution
discipline that must be enforced by review rather than by the compiler.

**Avro** is right for data pipelines and event logs where records outlive the code
that wrote them, because reader/writer resolution is designed for exactly that. It
is wrong for request-response APIs, where the registry dependency buys nothing.

What would change any of these: volume, retention, and who controls the consumers.
Not benchmark tables. If the data is read by people or by clients you do not
deploy, readability usually wins; if it is high-volume machine-to-machine and
retained, schemas win.

## Failure modes & operational cost

- **64-bit integers through JavaScript.** Silent, off-by-small-numbers ID
  corruption. The most common serialisation bug in practice.
- **Money as a float.** Rounding errors that accumulate and cannot be reconciled.
- **Base64 bloat.** A 33% size increase plus CPU, often on the hot path.
- **Protobuf field-number reuse.** Silent misinterpretation by older readers.
- **Strict parsers rejecting unknown fields.** Turns every additive producer change
  into an outage.
- **Losing the schema.** Protobuf or Avro data whose schema is not versioned and
  retained somewhere durable becomes undecodable. The registry is a hard
  dependency, and its availability is your data's availability.
- **Large-payload serialisation as a blocking cost.** JSON encoding is
  synchronous and proportional to size, which is a
  [blocked event loop](blocking-the-event-loop.md) waiting to happen.
- **Deserialising untrusted input into objects.** Format-dependent, and in some
  languages a remote-code-execution class — Python's `pickle` and YAML's default
  loader are the standard examples. Never accept those from outside.
- **Unbounded input size.** A parser that will happily allocate for a 2 GB body is
  a denial-of-service vector regardless of format.

## Open questions / to verify

Not checked against primary sources in this session:

- Actual size and encode/decode times for representative payloads here — JSON vs
  MessagePack vs protobuf, at 1 KB, 100 KB, 5 MB. Every ordering claim above should
  be replaced by measurements on real shapes.
- Whether the protobuf implementations in use preserve unknown fields on
  round-trip (proto3 behaviour changed historically, and it differs by language).
- Whether `orjson` / `ujson` change the precision behaviour for large integers, or
  only the speed.
- Whether the JSON parsers in use enforce a maximum depth or size by default.
- Avro's exact reader/writer resolution rules for a field added without a default.
- Whether `Content-Type` negotiation in the frameworks here can serve both JSON and
  a binary format from one handler without duplicating the serialisation layer.

Candidate for `practiced`: take one real response shape, serialise it four ways,
and record size and time at three payload sizes. Then break compatibility on
purpose — add a field, remove a field, reuse a protobuf number — and record which
combinations fail loudly versus silently.

## Sources

- [RFC 8259 — JSON](https://www.rfc-editor.org/rfc/rfc8259)
- [RFC 8949 — CBOR](https://www.rfc-editor.org/rfc/rfc8949)
- [Protocol Buffers — language guide and evolution rules](https://protobuf.dev/programming-guides/proto3/)
- [Apache Avro — specification, schema resolution](https://avro.apache.org/docs/current/specification/)
- [IEEE 754 double precision, and why 2^53 matters](https://docs.python.org/3/tutorial/floatingpoint.html)
