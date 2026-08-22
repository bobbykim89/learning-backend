---
title: Time, timezones, and clocks
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

Three distinct concepts that share the word "time", and mixing them is the source
of nearly every date bug:

1. **An instant** — a point on the universal timeline. Unambiguous. Represented as
   epoch seconds, or as UTC.
2. **A local date-time** — "2026-11-01 01:30" with no offset. Not an instant: it
   may denote two instants, one instant, or none, depending on the zone.
3. **A duration** — elapsed time, which is a different measurement problem
   entirely, because the clock you use to measure it can move.

### Two clocks, and only one of them measures duration

**Wall clock** (`time.time()`, `Date.now()`) reports civil time and is
**adjustable**: NTP corrects it, operators change it, virtual machines resume with
it stale. It can jump backwards. Subtracting two wall-clock readings can therefore
yield a negative duration, or one inflated by however much NTP corrected.

**Monotonic clock** (`time.monotonic()`, `process.hrtime.bigint()`) only ever
moves forward, has no relationship to civil time, and is meaningless as an
absolute value. It is the only correct clock for measuring how long something took.

The rule is short: **timestamps from the wall clock, durations from the monotonic
clock.** Every latency metric computed with `Date.now()` differences is subtly
wrong, and occasionally spectacularly wrong.

### Timezones are political, not arithmetic

An offset is not a timezone. `-05:00` is an offset; `America/New_York` is a zone,
which is a *history* of offsets, including future changes not yet legislated.
Governments change these with weeks of notice.

This produces the most consequential storage rule in the topic:

- **For an instant that already happened** — store UTC. Converting to UTC is
  lossless and the answer never changes.
- **For a future local event** — "the meeting is at 09:00 on 2027-03-15 in Berlin"
  — storing UTC is **wrong**. If Germany changes its DST rules before then, your
  stored instant now denotes 08:00 or 10:00 local. Store the local date-time plus
  the zone identifier, and compute the instant when you need it.

Nearly all "store everything as UTC" advice omits that exception, and calendars
that got it wrong is why.

### DST creates times that do not exist and times that happen twice

At a spring-forward transition, local clocks skip an hour: 02:30 simply never
occurs. At autumn's fall-back, 01:30 occurs twice, and a naive local time is
genuinely ambiguous.

Consequences that appear in production:

- A cron job scheduled at 02:30 local **does not run** on the spring day, and runs
  **twice** on the autumn day.
- Aggregating "hours yesterday" gives 23 or 25 hours, not 24, so a query grouping
  by hour has a missing or duplicated bucket.
- A validation rule that accepts any local time accepts times that do not exist.

## Why it exists (what came before)

Local solar time was genuinely local until railways made it untenable — schedules
require agreement across distance, so standard time zones were introduced for the
railroads' benefit, then legislated. Daylight saving was added as wartime and
energy policy, and is still adjusted or abolished by individual legislatures.

Computing inherited this. Unix chose to count seconds from an epoch in UTC, which
is why an instant is simple, and delegated the political mess to the **IANA time
zone database** — a continuously updated dataset of every jurisdiction's offset
history. That database is a *dependency*: a container image built last year has
last year's rules, and will convert future dates incorrectly for any zone that has
since changed.

Leap seconds are the remaining wrinkle: UTC is occasionally adjusted to track
Earth's rotation, which means a minute with 61 seconds. Most infrastructure now
"smears" the adjustment over hours rather than implementing it, because monotonic
assumptions in software are more fragile than the astronomy.

## Smallest example

```python
import time
from datetime import datetime, timezone, timedelta
from zoneinfo import ZoneInfo

# --- Durations: monotonic, always ---
t0 = time.monotonic()
time.sleep(0.05)
print(f"elapsed {time.monotonic() - t0:.3f}s")     # never negative, never NTP-skewed

# --- Instants: aware UTC, never naive ---
now = datetime.now(timezone.utc)                   # correct
# datetime.utcnow()  -> deprecated: returns a NAIVE datetime holding UTC,
#                       which then compares and arithmetics incorrectly.

# Naive and aware do not mix:
try:
    datetime.now() - now
except TypeError as e:
    print("TypeError:", e)

# --- The nonexistent local time ---
tz = ZoneInfo("America/New_York")
gap = datetime(2026, 3, 8, 2, 30, tzinfo=tz)       # 02:30 never occurs that day
print(gap, "->", gap.astimezone(timezone.utc))     # silently resolved, not rejected

# --- The ambiguous local time, disambiguated by fold ---
amb = datetime(2026, 11, 1, 1, 30, tzinfo=tz)
print(amb.replace(fold=0).astimezone(timezone.utc))   # first 01:30 (EDT)
print(amb.replace(fold=1).astimezone(timezone.utc))   # second 01:30 (EST)

# --- Future local event: store local + zone, not UTC ---
meeting = {"local": "2027-03-15T09:00:00", "zone": "Europe/Berlin"}
instant = datetime.fromisoformat(meeting["local"]).replace(
    tzinfo=ZoneInfo(meeting["zone"])).astimezone(timezone.utc)
print("computed at read time:", instant)
```

```javascript
// Duration: monotonic
const t0 = process.hrtime.bigint();
// ... work ...
console.log(`elapsed ${Number(process.hrtime.bigint() - t0) / 1e6}ms`);

// Date is an instant (ms since epoch, UTC) with locale-dependent formatting.
const d = new Date("2026-11-01T05:30:00Z");
console.log(d.toISOString());                                   // stable, UTC
console.log(d.toLocaleString("en-US", { timeZone: "America/New_York" }));

// The trap: parsing a date-only string is UTC; a date-time without offset is LOCAL
console.log(new Date("2026-03-15").toISOString());        // treated as UTC midnight
console.log(new Date("2026-03-15T00:00:00").toISOString()); // treated as LOCAL
```

That final pair is a genuine footgun: the same date, two parsing rules, differing
by your server's zone.

## Storing time

| Need | Store | Why |
|---|---|---|
| Past/current instant | `timestamptz` (UTC), or epoch integer | Lossless, unambiguous, sortable |
| Future local event | local date-time **+** IANA zone id | Offsets may change before it arrives |
| Date with no time | `date` | A birthday is not an instant; converting it invents one |
| Duration | integer + explicit unit | Named `timeout_ms`, not `timeout` |

On Postgres specifically, the naming misleads: `timestamptz` does **not** store a
zone. It converts the input to a UTC instant and discards the zone, then renders it
in the session's zone on read. `timestamp` stores exactly what you typed with no
zone at all — which is almost never what an application wants.

Serialise instants as RFC 3339 with an explicit offset (`2026-08-17T04:12:00Z`).
Those strings sort lexicographically in the same order as the instants, provided
the offset is always `Z` — mixed offsets break that property, which quietly breaks
sorting anything that compares them as strings.

## Tradeoffs & when it's wrong

**Epoch integers versus ISO strings.** Integers are compact, unambiguous, and
trivially comparable; they are unreadable in a log and carry no precision
indication, so seconds and milliseconds get confused. ISO strings are
self-describing and debuggable at the cost of size and parsing. For APIs read by
humans, strings; for internal high-volume records, integers with the unit in the
field name.

**Storing UTC for everything.** Simple and right for the overwhelming majority,
and silently wrong for future scheduled local events. The cost of doing it properly
is carrying a zone identifier and computing instants at read time.

**Converting to the user's zone in the backend versus the client.** Backend
conversion gives consistent rendering and needs an up-to-date tz database
server-side; client conversion offloads it and relies on the client's zone being
correct, which for a shared or server-rendered view it may not be.

What would change these: whether the time is in the past, and whether a human ever
reads the raw value.

## Failure modes & operational cost

- **Durations measured with the wall clock.** Negative latencies in metrics,
  timeouts that fire early or never, and rate limiters that can be defeated by an
  NTP correction.
- **Naive/aware mixing.** In Python, a `TypeError` at best; in code paths that
  coerce, a silent offset error.
- **`utcnow()`.** Returns a naive value that *looks* like UTC, so it compares
  incorrectly against aware values and serialises without an offset.
- **Cron at a DST boundary.** Jobs skipped or double-run once per year. Schedule
  UTC, or make the job idempotent — which is the more robust fix.
- **Stale tz database in a container.** Conversions for zones that changed rules
  are wrong, and nothing errors. Rebuild images or mount the host's zoneinfo.
- **Server zone dependence.** Code that behaves differently on a machine set to a
  non-UTC zone. Set containers to UTC and never rely on the default.
- **Date-only values turned into instants.** A birthday stored as midnight UTC
  becomes the previous day for anyone west of UTC.
- **Grouping by hour or day across a DST transition.** Missing or duplicated
  buckets, in reports that nobody reconciles.
- **Mixed offsets in stored strings.** Sorting breaks, and equality comparisons of
  the same instant written two ways fail.
- **Assuming the clock is monotonic across a VM suspend.** It is not, for the wall
  clock, and long sleeps can wake far later than intended.

## Open questions / to verify

Not checked against primary sources in this session:

- Whether `datetime.utcnow()` is formally deprecated in the Python version in use,
  and what the recommended replacement's exact form is.
- Whether `zoneinfo` falls back to the `tzdata` package or the system database on
  this platform, and which one the deployed container actually has.
- Whether the containers used here are set to UTC, and whether their zoneinfo is
  current.
- The status of the JavaScript `Temporal` API in the Node version in use, since it
  is the intended replacement for `Date`'s design problems.
- How the database driver in use converts `timestamptz` — whether it returns an
  aware datetime, and in which zone.
- Whether the job scheduler in this stack interprets its schedule in UTC or in a
  local zone, and what it does on the two DST days.

Candidate for `practiced`: schedule a job at 02:30 in a DST-observing zone, run the
clock through both transitions in a test, and demonstrate the skip and the
double-run. Then make the job idempotent and show both transitions become harmless.

## Sources

- [RFC 3339 — Date and Time on the Internet](https://www.rfc-editor.org/rfc/rfc3339)
- [IANA Time Zone Database](https://www.iana.org/time-zones)
- [Python `zoneinfo`](https://docs.python.org/3/library/zoneinfo.html) and
  [`time` — clocks](https://docs.python.org/3/library/time.html)
- [PostgreSQL — date/time types](https://www.postgresql.org/docs/current/datatype-datetime.html)
- [TC39 Temporal proposal](https://tc39.es/proposal-temporal/docs/)
