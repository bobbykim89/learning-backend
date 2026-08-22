---
title: Identifier strategies
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

How you name rows, and the three properties that conflict:

1. **Index locality** — do new rows land next to each other in the index, or
   scattered across it?
2. **Generation independence** — can a client, or any node, mint an ID without
   asking a central authority?
3. **Opacity** — does the ID reveal anything: existence of neighbours, volume,
   creation time?

No scheme maximises all three, which is why this is a decision rather than a
default.

| Scheme | Size | Locality | Generate offline | Leaks |
|---|---|---|---|---|
| `bigserial` / auto-increment | 8 bytes | excellent | no | neighbours, volume |
| UUIDv4 | 16 bytes | **poor** | yes | nothing |
| UUIDv7 / ULID | 16 bytes | good | yes | creation time |
| Snowflake-style | 8 bytes | good | yes, with coordination | time, node, rate |

### Why random IDs hurt write performance

This is the part that is usually stated as folklore, so here is the mechanism.

A B-tree index keeps entries in sorted order across fixed-size pages. With a
monotonically increasing key, every insert goes to the **rightmost** page: that
page stays in memory, it fills up, it splits once, and you move on. One hot page,
sequential writes, minimal I/O.

With a random key, each insert targets a random page. That page is probably not in
the buffer pool, so it must be read from disk to be modified. It is probably
partially full, so inserting may split it. Splits leave pages half-empty, so the
index grows larger than the data warrants, which makes the cache hit rate worse,
which causes more reads. The write-ahead log also grows, because full-page writes
get logged.

So a UUIDv4 primary key on a large, write-heavy table costs more I/O, a bigger
index, and a worse cache hit rate than a sequential key — and the effect appears
only once the index no longer fits in memory, which is to say in production and not
in development.

**UUIDv7 and ULID fix precisely this** by putting a timestamp in the high bits and
randomness in the low bits. Sorted by value means sorted roughly by creation time,
so inserts are local again while remaining unguessable and independently
generable. That is why they exist and why they are usually the right modern
default.

The cost is the leak: anyone holding one of your IDs learns when the row was
created, and holding two learns your creation rate.

### Enumeration, and what it is not

Sequential IDs let an attacker walk your data: if `/invoices/1041` is yours,
`/invoices/1042` is somebody's. This is real, and it is the reason public-facing
identifiers are usually opaque.

But be precise about the fix. **Opaque IDs are not authorisation.** A system that
relies on unguessable identifiers for access control has an
[insecure direct object reference](../03-security/authorization/broken-object-level-authorization.md)
with a longer key. The authorisation check on every access is the control; opacity
is defence in depth that buys you time and reduces casual scraping.

A common and good arrangement is **two identifiers**: an internal sequential
integer for keys, joins, and foreign keys, and an external opaque identifier for
URLs and APIs. You get locality where it affects performance and opacity where it
affects exposure, at the cost of one extra indexed column and the discipline never
to leak the internal one.

## Why it exists (what came before)

Auto-increment was the obvious answer when a database was one machine: the server
holds a counter, hands out the next value, done. Small, ordered, and free.

It breaks on three things. Distributed writes need coordination for that counter.
Offline or client-side creation cannot wait for a round trip. And merging two
datasets that both used counters produces collisions on every row.

UUIDs solved all three by making IDs large and random enough that coordination
becomes unnecessary — a probabilistic answer to a structural problem. Version 4 is
just 122 random bits, which is why it has no locality at all.

UUIDv7 is the reconciliation, standardised in RFC 9562 alongside v6 and v8, after
ULID and Snowflake demonstrated in practice that a time prefix recovers locality
without giving up independent generation.

## Smallest example

```python
import uuid, time, os
from datetime import datetime, timezone

# v4: 122 random bits. No order, no information, no locality.
print(uuid.uuid4())

# A minimal UUIDv7-shaped id: 48-bit ms timestamp, then randomness.
# (Prefer a library implementation; this is to show the structure.)
def uuid7() -> uuid.UUID:
    ms = int(time.time() * 1000) & ((1 << 48) - 1)
    rand_a = int.from_bytes(os.urandom(2), "big") & 0x0FFF
    rand_b = int.from_bytes(os.urandom(8), "big") & ((1 << 62) - 1)
    val = (ms << 80) | (0x7 << 76) | (rand_a << 64) | (0b10 << 62) | rand_b
    return uuid.UUID(int=val)

ids = [uuid7() for _ in range(3)]
for i in ids:
    print(i, "version", i.version)
print("sorted by value == created in order:", ids == sorted(ids))   # True

# The timestamp is recoverable — this is the leak, by design
first = int(str(ids[0]).replace("-", "")[:12], 16)
print("created at:", datetime.fromtimestamp(first / 1000, timezone.utc))
```

```javascript
import { randomUUID } from "node:crypto";

console.log(randomUUID());   // v4

// Storage matters as much as generation: 16 bytes vs 36 characters
const id = randomUUID();
console.log(id.length);                              // 36 chars as text
console.log(Buffer.from(id.replace(/-/g, ""), "hex").length);  // 16 bytes binary
```

```sql
-- Storage: use the native type, never varchar(36)
CREATE TABLE orders (
  id          uuid PRIMARY KEY,        -- 16 bytes
  public_id   text UNIQUE,             -- optional opaque external handle
  created_at  timestamptz NOT NULL DEFAULT now()
);

-- The anti-pattern, roughly 2-3x the bytes per key and slower comparisons:
-- id varchar(36) PRIMARY KEY
```

## Tradeoffs & when it's wrong

**Auto-increment** is right for internal tables, single-writer databases, and
anything where locality dominates and IDs never appear in URLs. It is wrong for
public identifiers, wrong under sharding without a coordination scheme, and wrong
when clients must create records offline.

**UUIDv4** is right when you want zero information leakage and zero coordination,
and when the table is small enough or read-heavy enough that index locality does
not matter. It is wrong as the primary key of a large write-heavy table — and
especially wrong in a database with **clustered** primary keys, such as MySQL's
InnoDB, where the table itself is stored in primary-key order, so random keys
scatter the *data* and not merely an index.

**UUIDv7 / ULID** is the sensible modern default for public identifiers: locality,
independence, and no enumeration. It is wrong where creation time is sensitive —
and that is not a hypothetical. Leaked IDs let a competitor measure your signup
rate, and in some products the timing of a record is itself confidential.

**Snowflake-style** is right at high volume where 8 bytes matters and you can
manage node identifiers. It is wrong when you cannot guarantee unique node IDs or a
sane clock, because both duplicate IDs and ID regression follow from getting those
wrong.

What would change the answer: table size and write rate, whether the ID is public,
whether creation time is sensitive, and whether the database clusters on the
primary key.

## Failure modes & operational cost

- **UUIDs stored as text.** Two to three times the bytes in every index and every
  foreign key, plus slower comparisons. Extremely common, and easy to fix only
  before the data exists.
- **Random UUID as a clustered primary key.** Page splits and write amplification
  that appear as unexplained I/O growth months after launch.
- **Opacity mistaken for authorisation.** IDOR with extra steps.
- **Timestamp leakage from v7/ULID.** Creation time and rate disclosure, sometimes
  a genuine business-confidentiality problem.
- **Exposing sequential IDs in URLs.** Scraping, plus volume disclosure: invoice
  `#4102` tells a competitor how many invoices exist.
- **Collision assumptions under duplicated node IDs.** Snowflake schemes fail
  silently when two nodes share an ID, producing duplicate keys that surface as
  constraint violations far from the cause.
- **Clock regression breaking monotonicity.** A backwards NTP step can make a
  time-prefixed generator emit an ID lower than one it already emitted, which
  breaks any code treating ID order as creation order — see
  [time and clocks](time-and-timezones.md).
- **ULID monotonicity within the same millisecond.** Not guaranteed unless the
  implementation explicitly provides a monotonic factory.
- **Changing scheme after launch.** Requires rewriting every foreign key and every
  externally shared reference. This is the decision to get right early, and the one
  cheapest to get right early.

## Open questions / to verify

Not checked against primary sources in this session:

- Whether the PostgreSQL version in use provides a built-in `uuidv7()` function, or
  whether generation must happen in the application or an extension.
- Whether the ORM in use maps a `uuid` column to the native 16-byte type or falls
  back to text.
- Whether MySQL's InnoDB clustered-index penalty for random keys is materially
  reduced by recent versions, or remains as described.
- Actual measured insert throughput and index size for v4 versus v7 on a table
  large enough to exceed the buffer pool here — every locality claim above should
  be replaced by that measurement.
- Whether the UUIDv7 implementations available in Python and Node guarantee
  monotonicity within a millisecond, and how they behave if the clock steps back.
- Whether any current API in this system leaks internal sequential IDs alongside
  public ones.

Candidate for `practiced`: create two tables of ten million rows, keyed by UUIDv4
and UUIDv7, with a buffer pool deliberately smaller than the indexes. Measure
insert throughput, final index size, and cache hit ratio for both, and report the
ratio.

## Sources

- [RFC 9562 — UUIDs (v1, v4, v6, v7, v8)](https://www.rfc-editor.org/rfc/rfc9562)
- [ULID specification](https://github.com/ulid/spec)
- [PostgreSQL — UUID type](https://www.postgresql.org/docs/current/datatype-uuid.html)
- [PostgreSQL — index internals, B-tree](https://www.postgresql.org/docs/current/btree.html)
