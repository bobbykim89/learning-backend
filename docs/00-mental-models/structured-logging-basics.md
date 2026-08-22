---
title: Structured logging from day one
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

Emitting logs as **machine-parseable events with stable keys** rather than as
prose. One JSON object per line, each carrying the same field names for the same
meanings.

```
# unstructured — readable by one human, queryable by nobody
Order 4102 failed for user bob after 2.3s: card declined

# structured — the same facts, addressable
{"ts":"2026-08-17T04:12:00Z","level":"error","msg":"order failed",
 "order_id":4102,"user_id":"u_88","duration_ms":2312,
 "reason":"card_declined","trace_id":"9af1...","service":"checkout"}
```

The difference only matters at the moment you need it, which is why it must be
done first. "How many orders failed with `card_declined` in the last hour, grouped
by region" is a query against the second and a regex-writing exercise against the
first. Worse, the first form makes *aggregation across instances* impossible,
which is the normal case as soon as you run more than one worker.

### Why `print` / `console.log` does not survive production

Six concrete reasons, each of which you hit in order:

1. **No severity.** You cannot lower the volume without deleting information, and
   you cannot alert on "errors" because nothing is marked as one.
2. **No context.** A line saying `timeout` does not say which request, which user,
   or which downstream call. In a concurrent server, interleaved lines from
   different requests cannot be separated at all.
3. **No correlation.** One user action crosses several services. Without a shared
   identifier there is no way to assemble the story.
4. **Unparseable.** Multi-line stack traces become several unrelated log entries.
   A message containing a newline or a quote breaks naive parsers.
5. **Unbounded cost.** Log volume grows with traffic, and ingestion is usually
   billed by volume. Without levels and sampling there is no lever to pull.
6. **It can block.** Writing synchronously to a slow destination stalls the caller
   — and on an event loop, stalls everything. See
   [blocking the event loop](blocking-the-event-loop.md).

### The fields worth standardising on day one

Not a long list, and the value comes from the names never changing:

- `ts` — RFC 3339, UTC, always
- `level` — and mean the same thing by each one, every time
- `msg` — a short, **constant** string. Put the variable parts in fields, so that
  `msg` remains groupable. `"order failed"` with an `order_id` field, never
  `"order 4102 failed"`.
- `service`, `version`, `env` — attached automatically, never by hand
- `trace_id` / `request_id` — the correlation key
- `duration_ms` — for anything with a duration, measured with a monotonic clock
- `error` — type, message, and stack as separate fields

The `msg`-stays-constant rule is the one most often broken and the one that most
determines whether logs are queryable. Interpolating an ID into the message
produces a million distinct messages and no way to count an occurrence class.

### Correlation IDs

Accept an inbound `traceparent` or request-ID header if present, generate one if
not, attach it to every log line for that request, and pass it to every downstream
call.

The mechanism that makes this bearable is implicit context, so you are not
threading an argument through every function:

- **Python** — `contextvars`, which are async-task-aware
- **Node** — `AsyncLocalStorage` from `node:async_hooks`

Both survive `await` boundaries, which a thread-local would not in an async
runtime.

## Why it exists (what came before)

Logs were files on a machine you could log into, read with `grep`, and rotate
nightly. That model works when there is one process on one host and a human
watching it.

Three changes broke it. Multiple instances meant the story of a single request was
split across hosts. Ephemeral compute meant the host — and its files — vanished,
often precisely when it crashed. And aggregation tooling arrived that could query
across everything, but only if the events were parseable.

Hence the twelve-factor rule: **treat logs as an event stream and write to
stdout**, letting the platform collect, ship, and retain them. Managing files and
rotation inside a container reimplements badly what the platform already does, and
loses the logs when the container dies.

Structured logging is the same shift applied to the line format: stop writing
sentences for a human with `grep`, start emitting records for a query engine.

## Smallest example

```python
import logging, json, time, uuid, contextvars
from datetime import datetime, timezone

request_id = contextvars.ContextVar("request_id", default=None)

class JsonFormatter(logging.Formatter):
    def format(self, record):
        payload = {
            "ts": datetime.now(timezone.utc).isoformat(timespec="milliseconds"),
            "level": record.levelname.lower(),
            "msg": record.getMessage(),          # constant string
            "logger": record.name,
            "service": "checkout",
            "request_id": request_id.get(),
        }
        if record.exc_info:                      # stack as a field, not extra lines
            payload["error"] = self.formatException(record.exc_info)
        payload.update(getattr(record, "extra_fields", {}))
        return json.dumps(payload)

handler = logging.StreamHandler()                # stdout; the platform collects it
handler.setFormatter(JsonFormatter())
log = logging.getLogger("checkout")
log.addHandler(handler)
log.setLevel(logging.INFO)

def handle_order(order_id):
    request_id.set(str(uuid.uuid4()))            # or from the inbound header
    t0 = time.monotonic()                        # monotonic, for durations
    try:
        raise ValueError("card declined")
    except ValueError:
        log.error("order failed", exc_info=True, extra={"extra_fields": {
            "order_id": order_id,
            "duration_ms": round((time.monotonic() - t0) * 1000, 1),
            "reason": "card_declined",
        }})

handle_order(4102)
```

```javascript
import { AsyncLocalStorage } from "node:async_hooks";
import { randomUUID } from "node:crypto";

const ctx = new AsyncLocalStorage();

function log(level, msg, fields = {}) {
  process.stdout.write(JSON.stringify({
    ts: new Date().toISOString(),
    level, msg,                                  // msg stays constant
    service: "checkout",
    request_id: ctx.getStore()?.requestId,
    ...fields,
  }) + "\n");
}

// Middleware: one context per request, id from the caller when supplied
function withRequestContext(req, res, next) {
  const requestId = req.headers["x-request-id"] ?? randomUUID();
  ctx.run({ requestId }, () => {
    const t0 = process.hrtime.bigint();
    res.on("finish", () => log("info", "request complete", {
      method: req.method,
      route: req.url,
      status: res.statusCode,
      duration_ms: Number(process.hrtime.bigint() - t0) / 1e6,
    }));
    next();
  });
}
```

Note what is *not* in either example: no log file, no rotation, no timestamps
formatted for human reading, and no variable data inside `msg`.

## Tradeoffs & when it's wrong

**JSON lines versus human-readable text.** JSON is queryable and machine-shippable;
it is also unpleasant to read in a terminal during local development. The usual
resolution is a pretty-printing renderer in development and JSON in every deployed
environment — with the important caveat that the two must not diverge in *content*,
or you will debug a format you never actually run.

**Logs versus metrics versus traces.** Logging every request gives complete detail
at a cost proportional to traffic. A metric gives you the rate and the percentile
for a fixed, tiny cost, and cannot tell you about one specific request. Traces sit
between. Logging what should have been a counter is the most common source of
runaway logging bills. The rule of thumb: if you only ever aggregate it, it is a
metric.

**Sampling.** Necessary at volume, and it will eventually discard the one request
you needed. Sample successes aggressively, keep all errors, and make the sampling
decision per trace rather than per line so a sampled trace is not half-missing.

**Structured from day one versus later.** Retrofitting is genuinely expensive —
every call site changes, and old logs remain unqueryable — which is the argument for
doing it first. The counter-argument is honest: for a small service with one
instance and no aggregation tooling, `print` is enough, and a JSON formatter is
ceremony. What decides it: whether you will ever run more than one instance, and
whether anyone will need to answer a question about yesterday.

## Failure modes & operational cost

- **Logging inside a loop.** One line per row over a large result set produces
  millions of lines and a bill; it is also usually the slowest part of the request.
- **Variable data in `msg`.** Destroys grouping and inflates cardinality in any
  system that indexes messages.
- **Unbounded field values.** Logging a whole request body or a stack of a
  megabyte. Truncate at the boundary.
- **PII and secrets in logs.** Tokens, card numbers, passwords, personal data —
  arriving via "log the whole object for debugging". Redact by allowlist, not by
  denylist, and remember that logs are retained and widely readable.
- **Log injection.** User input containing newlines can forge log entries. Encoding
  as JSON handles it; string concatenation does not.
- **Blocking on a slow sink.** Synchronous writes to a network destination stall
  the request path. Buffer and flush asynchronously.
- **Losing logs on crash.** Buffered output not flushed at exit means the lines
  explaining the crash are the ones you lose. Flush on fatal paths.
- **Log-and-throw.** The same error logged at three levels of the stack, tripling
  volume and making the count meaningless.
- **Different formats per environment.** The parser works in staging and fails in
  production.
- **Level discipline drift.** Once `error` is used for things that are not errors,
  alerting on it becomes impossible and everyone stops looking.

## Open questions / to verify

Not checked against primary sources in this session:

- Whether the logging library in use here writes synchronously to stdout, and
  whether stdout is a pipe or a file in the deployed environment — pipes have
  different blocking behaviour when the reader is slow.
- Whether Python's `QueueHandler` / `QueueListener` is the current recommendation
  for non-blocking logging, and whether it drops or blocks when the queue fills.
- Whether `contextvars` propagate correctly through the thread-pool escape hatches
  used for blocking calls.
- Whether the tracing setup in use emits `traceparent` in W3C format, so that
  `trace_id` in logs joins to spans.
- What the log ingestion cost per gigabyte and the retention window actually are,
  since those numbers decide the sampling policy.
- Whether the platform truncates lines above a length, which silently destroys
  large entries.

Candidate for `practiced`: add structured logging with correlation IDs to a
two-service call path, then answer a question that requires joining them — "for the
slowest 1% of checkout requests yesterday, what did the payment service return" —
using only queries against the logs. Then log a stack trace inside a loop and
measure the throughput cost of the logging alone.

## Sources

- [The Twelve-Factor App — Logs](https://12factor.net/logs)
- [Python `logging` — cookbook, including QueueHandler](https://docs.python.org/3/howto/logging-cookbook.html)
- [Python `contextvars`](https://docs.python.org/3/library/contextvars.html)
- [Node `AsyncLocalStorage`](https://nodejs.org/api/async_context.html)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
