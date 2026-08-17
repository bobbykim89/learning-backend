# Repository Scaffold Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold `learn-backend` — `.gitignore`, `CLAUDE.md`, and a `docs/` tree of phase folders each carrying a README that lists its topics as links to planned filenames — so that every future topic doc has a settled home and shape.

**Architecture:** The root `README.md` is the curriculum and the taxonomy; the `docs/` tree mirrors it exactly. Twelve phase folders (`00-mental-models` … `11-advanced`) plus an unnumbered `capstones/`. Phase 2 and Phase 3 get subsection subfolders mirroring their `###` headings; the other ten phases stay flat. No topic files are created — folder READMEs link to filenames that will exist once studied, and those links are the naming contract.

**Tech Stack:** Markdown only. No language toolchain, no dependencies, no build step. Verification is `ls`, `grep`, and `git`.

**Spec:** [docs/superpowers/specs/2026-08-16-repo-scaffold-design.md](../specs/2026-08-16-repo-scaffold-design.md)

## Testing Adaptation — Read This First

This repository contains no executable code, so there is no test framework and
nothing to unit test. TDD's loop is preserved in its verifiable form:

1. Run the verification command **first** and observe it fail.
2. Create the files.
3. Run the same command again and observe it pass.
4. Commit.

Every task below supplies the exact command and the exact expected output for
both runs. Do not skip the first run — a check you never saw fail is a check you
have not validated.

Run all commands from the repository root, `/mnt/projects/learn-backend`.

## Global Constraints

- **Repository is documentation only.** Code examples live inline in `.md` files as fenced blocks. Never add a `package.json`, `pyproject.toml`, `requirements.txt`, lockfile, or any script.
- **Filename convention.** Kebab-case slug derived from the topic bullet, trimmed to the core concept, no number prefix, five words or fewer, `.md` extension. Matches `^[a-z0-9]+(-[a-z0-9]+)*\.md$`.
- **Phase folder names** carry an `NN-` prefix so they sort in curriculum order. `capstones/` has no prefix because it is not a phase.
- **Every bullet maps to exactly one file, in exactly one folder.** Two bullets that cover adjacent ground (Phase 2's `idempotency-keys.md` and Phase 5's `api-idempotency.md`) get distinct filenames and cross-link; they are never merged.
- **Link text in folder READMEs is the bullet text verbatim** from the root `README.md`, with the `- [ ] ` checkbox prefix removed and nothing else changed. No rewording, no shortening.
- **Status vocabulary** is exactly `not-started`, `learning`, `practiced`. Nothing else.
- **Every folder and subsection subfolder contains a `README.md`.** A folder with no file cannot be committed to git, so folders come into existence with their README, never as bare directories or `.gitkeep`.
- **Do not create topic `.md` files.** Folder READMEs link to files that do not exist yet. That is intended, not a bug to fix.
- **Only phase-level READMEs link to `CLAUDE.md`,** at `../../CLAUDE.md`. Subsection READMEs (Phase 2 and Phase 3) do not link it at all — from three levels deep the relative path differs, and the link adds nothing their parent README does not already carry.
- **Do not edit the root `README.md`** except for the one `sed` in Task 11. Its wording is the curriculum and stays byte-identical apart from checkbox removal.

---

### Task 1: .gitignore

**Files:**
- Create: `.gitignore`

**Interfaces:**
- Consumes: nothing.
- Produces: `scratch/` becomes the documented throwaway location, referenced by `CLAUDE.md` in Task 2.

- [ ] **Step 1: Run the verification command and watch it fail**

Run:

```bash
test -f .gitignore && echo PRESENT || echo ABSENT
```

Expected: `ABSENT`

- [ ] **Step 2: Create `.gitignore` with exactly this content**

```gitignore
# OS
.DS_Store
Thumbs.db

# Editors
.vscode/
.idea/
*.swp

# Secrets
.env
.env.*

# Local scratch
scratch/

# Claude local settings
.claude/settings.local.json

# Subagent-driven development workspace
.superpowers/
```

- [ ] **Step 3: Run the verification command again**

Run:

```bash
test -f .gitignore && echo PRESENT || echo ABSENT
grep -c . .gitignore
```

Expected: `PRESENT` then `16` (non-blank lines: 6 comments plus 10 patterns).

- [ ] **Step 4: Confirm no language toolchain patterns crept in**

Run:

```bash
grep -nE 'node_modules|__pycache__|\.venv|dist|target' .gitignore || echo CLEAN
```

Expected: `CLEAN`. This repository holds no code, so those patterns would be noise. If they appear, delete them.

- [ ] **Step 5: Commit**

```bash
git add .gitignore
git commit -m "chore: add .gitignore for docs-only repository"
```

---

### Task 2: CLAUDE.md

**Files:**
- Create: `CLAUDE.md`

**Interfaces:**
- Consumes: `scratch/` from Task 1.
- Produces: the topic template and status vocabulary that every future topic doc follows. No later task in this plan depends on it, but every future study session does.

- [ ] **Step 1: Run the verification command and watch it fail**

Run:

```bash
test -f CLAUDE.md && echo PRESENT || echo ABSENT
```

Expected: `ABSENT`

- [ ] **Step 2: Create `CLAUDE.md` with exactly this content**

````markdown
# CLAUDE.md

## What this repository is

A personal study repository for backend engineering, worked through with AI
assistance. It contains **documentation only** — no runnable projects, no
package manifests, no build step. Code examples live inline in `.md` files as
fenced blocks.

The root `README.md` is the curriculum: twelve phases (Phase 0–11) plus six
capstone projects, 265 topic bullets, each scoped to one 30–90 minute study
session. The `docs/` tree mirrors that curriculum exactly.

Capstone project code lives in separate repositories. `capstones/` here holds
notes and design decisions only.

## Structure

```
docs/
  README.md                     index of phases
  00-mental-models/
  01-architecture/
  02-data/                      has subsection subfolders
  03-security/                  has subsection subfolders
  04-caching-and-performance/
  05-api-design/
  06-async-and-distributed/
  07-reliability-and-operations/
  08-infrastructure-and-delivery/
  09-testing-and-quality/
  10-scaling/
  11-advanced/
  capstones/
  superpowers/                  meta: specs and plans, not curriculum
```

Every folder has a `README.md` listing its topics as links. A link may point at
a file that does not exist yet — that is the naming contract for a topic not yet
studied, not a broken link to fix.

Only Phase 2 and Phase 3 nest, because only they have `###` subsections in the
root README. Do not invent subsections for the other phases.

## Filename convention

Kebab-case slug of the topic bullet, trimmed to the core concept, no number
prefix, five words or fewer.

| Bullet | Filename |
|---|---|
| How B-tree indexes work; why an index is a sorted copy with a cost | `btree-indexes.md` |
| CAP and PACELC — stated precisely, not as a slogan | `cap-and-pacelc.md` |

One bullet, one file, one folder. Before creating a topic file, check the
relevant folder README — the filename is already chosen there.

## Topic template

Every topic doc uses this frontmatter and these sections, in this order:

```markdown
---
title: <topic name>
phase: <phase folder name>
status: not-started | learning | practiced
updated: YYYY-MM-DD
---

## What it is

## Why it exists (what came before)

## Smallest example

## Tradeoffs & when it's wrong

## Failure modes & operational cost

## Open questions / to verify

## Sources
```

`Smallest example` carries Python and TypeScript/Node snippets where both apply.

Leave a section empty rather than padding it. An empty **Tradeoffs** or **Open
questions** section is useful signal that the topic is not finished.

## Status

`status:` in the frontmatter is the single source of truth for progress. The root
README is a curriculum outline, not a tracker — do not add checkboxes back to it.

- `not-started` — file exists as a stub only
- `learning` — being read and written up
- `practiced` — something was built with the concept

`practiced` requires having built something. Reading about a topic does not earn it.

At the end of a session, update `status` and set `updated` to today's date.

## How to teach here

The failure mode of AI-assisted learning is fluent summaries mistaken for
understanding. Accordingly:

- **Argue both sides of a tradeoff before recommending anything.** For any
  decision (JWTs vs sessions, ORM vs raw SQL), make the strongest case for each
  option, then name the questions about the system that decide it.
- **Failure modes and operational cost come before endorsement.** Never
  recommend a tool or pattern without stating what it costs to run and how it
  breaks.
- **Flag version-specific claims as unverified.** Library APIs, defaults, and
  flags drift. Any concrete claim about a version belongs in **Open questions /
  to verify** unless it was checked against primary docs in this session.
- **Quiz with scenarios, not definitions,** and do not reveal the answer until
  an answer has been committed to.
- **Say "I don't know."** A gap named is worth more than a confident paragraph.

## Conventions

- Prose in docs is plain and specific. No filler, no marketing register.
- Cite sources with links in the **Sources** section; prefer primary docs and
  papers over blog posts.
- `scratch/` is gitignored — put throwaway experiments there.
- Never add a language toolchain to this repository.
````

- [ ] **Step 3: Run the verification command again**

Run:

```bash
test -f CLAUDE.md && echo PRESENT || echo ABSENT
grep -c '^status\|^- `not-started`\|^- `learning`\|^- `practiced`' CLAUDE.md
```

Expected: `PRESENT` then `4` — the three status bullets plus the `status:` line
inside the frontmatter of the topic template, which also starts at column 0.

- [ ] **Step 4: Confirm the template's seven sections are all present**

Run:

```bash
grep -c '^## What it is$\|^## Why it exists\|^## Smallest example$\|^## Tradeoffs\|^## Failure modes\|^## Open questions\|^## Sources$' CLAUDE.md
```

Expected: `7`

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md with repo conventions and topic template"
```

---

### Task 3: docs/README.md index

**Files:**
- Create: `docs/README.md`

**Interfaces:**
- Consumes: nothing.
- Produces: the twelve phase folder names plus `capstones/`, exactly as Tasks 4–10 must create them.

The one-line description under each phase is that phase's own subtitle from the
root `README.md`, verbatim where one exists. Phases 4–11 have no subtitle in the
root README; the descriptions below are supplied by this plan — use them as written.

- [ ] **Step 1: Run the verification command and watch it fail**

Run:

```bash
test -f docs/README.md && echo PRESENT || echo ABSENT
```

Expected: `ABSENT`

- [ ] **Step 2: Create `docs/README.md` with exactly this content**

```markdown
# Documentation

One file per topic, mirroring the curriculum in the [root README](../README.md).
Progress lives in each topic file's `status:` frontmatter — see
[CLAUDE.md](../CLAUDE.md) for the template and conventions.

A link to a file that does not exist yet means the topic has not been studied.

## Phases

- [Phase 0 — Mental Models](00-mental-models/README.md) — what actually happens when a request arrives
- [Phase 1 — Application Architecture and Code Design](01-architecture/README.md) — where dependency injection, testability, and "how do I structure this" live
- [Phase 2 — Data: Relational Depth First](02-data/README.md) — the single highest-leverage area; most backend performance problems are database problems
- [Phase 3 — Security](03-security/README.md) — a design constraint that shapes every layer, not a checklist to bolt on
- [Phase 4 — Caching and Performance](04-caching-and-performance/README.md) — the cache hierarchy, invalidation, and measuring before optimising
- [Phase 5 — API Design and Contracts](05-api-design/README.md) — resources, pagination, versioning, realtime, and webhooks
- [Phase 6 — Asynchronous and Distributed Systems](06-async-and-distributed/README.md) — queues, delivery semantics, and consistency
- [Phase 7 — Reliability and Operations](07-reliability-and-operations/README.md) — failure handling, observability, and SLOs
- [Phase 8 — Infrastructure and Delivery](08-infrastructure-and-delivery/README.md) — containers, orchestration, IaC, and CI/CD
- [Phase 9 — Testing and Quality](09-testing-and-quality/README.md) — test strategy, integration testing, and static analysis
- [Phase 10 — Scaling](10-scaling/README.md) — replicas, sharding, rate limiting, and multi-region
- [Phase 11 — Advanced and Specialized](11-advanced/README.md) — runtime and database internals, and specialised workloads

## Capstones

- [Capstone projects](capstones/README.md) — notes and design decisions; the code lives in separate repositories

## Meta

- [superpowers/specs/](superpowers/specs/) — design docs
- [superpowers/plans/](superpowers/plans/) — implementation plans
```

- [ ] **Step 3: Run the verification command again**

Run:

```bash
test -f docs/README.md && echo PRESENT || echo ABSENT
grep -c '^- \[Phase ' docs/README.md
```

Expected: `PRESENT` then `12`

- [ ] **Step 4: Commit**

```bash
git add docs/README.md
git commit -m "docs: add docs/ index of phases"
```

---

### Task 4: Phase 0 and Phase 1 folder READMEs

**Files:**
- Create: `docs/00-mental-models/README.md`
- Create: `docs/01-architecture/README.md`

**Interfaces:**
- Consumes: folder names from Task 3.
- Produces: 17 + 21 topic filenames. Nothing later depends on them.

Each README follows this shape. Link text is the bullet verbatim from the root
`README.md`; the link target is the slug from the table below.

```markdown
# Phase 0 — Mental Models: What Actually Happens When a Request Arrives

The layer most CRUD experience skips entirely. Everything later depends on this.

Files are created when the topic is studied — see [CLAUDE.md](../../CLAUDE.md).

- [The lifecycle of a request: DNS → TCP → TLS → HTTP → your handler → response](request-lifecycle.md)
- ...
```

The heading and the italic-free subtitle line come from the root README verbatim
(lines 10 and 12 for Phase 0; lines 34 and 36 for Phase 1).

- [ ] **Step 1: Run the verification command and watch it fail**

Run:

```bash
ls docs/00-mental-models/README.md docs/01-architecture/README.md 2>&1 | tail -2
```

Expected: two "No such file or directory" errors.

- [ ] **Step 2: Create `docs/00-mental-models/README.md`**

Heading: `# Phase 0 — Mental Models: What Actually Happens When a Request Arrives`
Subtitle: `The layer most CRUD experience skips entirely. Everything later depends on this.`

17 bullets, in root README order (lines 14–30), with these targets:

| Bullet (link text, verbatim from root README) | Target |
|---|---|
| The lifecycle of a request: DNS → TCP → TLS → HTTP → your handler → response | `request-lifecycle.md` |
| Process vs thread vs coroutine; what "concurrency" vs "parallelism" actually means | `process-thread-coroutine.md` |
| CPU-bound vs I/O-bound work, and why the answer changes your entire architecture | `cpu-bound-vs-io-bound.md` |
| Python's execution model: the GIL, `asyncio` event loop, when threads still help, `multiprocessing` | `python-execution-model.md` |
| Node's execution model: libuv, event loop phases, the microtask queue, worker threads | `node-execution-model.md` |
| WSGI vs ASGI (Gunicorn, Uvicorn, workers vs threads); Node clustering | `wsgi-asgi-and-clustering.md` |
| Blocking the event loop: how to cause it, how to detect it, how to fix it in both runtimes | `blocking-the-event-loop.md` |
| Sockets, file descriptors, and why "too many open files" happens | `sockets-and-file-descriptors.md` |
| TCP fundamentals: handshake, keep-alive, connection reuse, head-of-line blocking | `tcp-fundamentals.md` |
| TLS: handshake, certificates, chain of trust, termination points, mTLS preview | `tls-fundamentals.md` |
| HTTP/1.1 vs HTTP/2 vs HTTP/3 — multiplexing, and what changes for your server | `http-versions.md` |
| HTTP semantics in depth: safe vs idempotent methods, status code selection, header taxonomy | `http-semantics.md` |
| Serialization formats: JSON, MessagePack, Protobuf, Avro — size, speed, schema evolution | `serialization-formats.md` |
| Text encoding, Unicode, normalization, and the bugs they cause | `text-encoding-and-unicode.md` |
| Time: UTC discipline, timezones, DST, monotonic vs wall clock, storing timestamps | `time-and-timezones.md` |
| Identifiers: auto-increment vs UUIDv4 vs UUIDv7/ULID — index locality and enumeration risk | `identifier-strategies.md` |
| Structured logging from day one (why `print`/`console.log` doesn't survive contact with prod) | `structured-logging-basics.md` |

- [ ] **Step 3: Create `docs/01-architecture/README.md`**

Heading: `# Phase 1 — Application Architecture and Code Design`
Subtitle: `Where dependency injection, testability, and "how do I structure this" live.`

21 bullets, in root README order (lines 38–58), with these targets:

| Bullet | Target |
|---|---|
| Layered architecture: transport → service/use-case → repository → data source | `layered-architecture.md` |
| Why the layers exist: dependency direction, and what breaks when you skip them | `why-layers-exist.md` |
| Inversion of Control and Dependency Injection — the concept, independent of frameworks | `inversion-of-control-and-di.md` |
| Manual/constructor DI in plain Python and plain TypeScript (start here, no library) | `manual-di.md` |
| DI in practice: FastAPI `Depends`, `dependency-injector`; NestJS, tsyringe, InversifyJS | `di-frameworks.md` |
| Service lifetimes: singleton vs scoped vs transient, and request-scoped state | `service-lifetimes.md` |
| DI tradeoffs: when a container is overkill, magic vs explicitness, startup cost | `di-tradeoffs.md` |
| Ports and adapters (hexagonal architecture); when it pays off and when it's ceremony | `ports-and-adapters.md` |
| SOLID, pragmatically — which principles matter daily, which are academic | `solid-pragmatically.md` |
| Composition over inheritance; protocols/structural typing (Python `Protocol`, TS interfaces) | `composition-and-protocols.md` |
| Domain modeling: entities, value objects, aggregates, invariants | `domain-modeling.md` |
| DTOs vs domain models vs DB rows — why collapsing them hurts later | `dtos-vs-domain-models.md` |
| Validation at the boundary: Pydantic v2, Zod — parse, don't validate | `boundary-validation.md` |
| Error design: exception hierarchies vs result types, expected vs exceptional failures | `error-design.md` |
| Error taxonomy for APIs: client vs server, retryable vs terminal, RFC 9457 problem details | `api-error-taxonomy.md` |
| Configuration and the 12-factor app; typed config, fail-fast on startup | `configuration-and-12-factor.md` |
| Feature flags as an architectural tool | `feature-flags.md` |
| Project structure: layer-first vs feature-first, the modular monolith | `project-structure.md` |
| When to split a service — and the strong case for _not_ doing microservices yet | `when-to-split-a-service.md` |
| Middleware/interceptor pipelines: cross-cutting concerns done right | `middleware-pipelines.md` |
| Writing an Architecture Decision Record (ADR) — capturing tradeoffs, not just choices | `architecture-decision-records.md` |

- [ ] **Step 4: Run the verification command again**

Run:

```bash
grep -c '^- \[' docs/00-mental-models/README.md docs/01-architecture/README.md
```

Expected:

```
docs/00-mental-models/README.md:17
docs/01-architecture/README.md:21
```

- [ ] **Step 5: Check every link target matches the filename convention**

Run:

```bash
grep -ohP '\]\(\K[^)]+' docs/00-mental-models/README.md docs/01-architecture/README.md \
  | grep -vP '^[a-z0-9]+(-[a-z0-9]+)*\.md$' | grep -v '^\.\./\.\./CLAUDE\.md$' || echo CLEAN
```

Expected: `CLEAN`

- [ ] **Step 6: Commit**

```bash
git add docs/00-mental-models/README.md docs/01-architecture/README.md
git commit -m "docs: add Phase 0 and Phase 1 topic indexes"
```

---

### Task 5: Phase 2 — Data

**Files:**
- Create: `docs/02-data/README.md`
- Create: `docs/02-data/relational-fundamentals/README.md`
- Create: `docs/02-data/performance/README.md`
- Create: `docs/02-data/transactions-and-concurrency/README.md`
- Create: `docs/02-data/orms-and-access-layers/README.md`
- Create: `docs/02-data/schema-evolution/README.md`
- Create: `docs/02-data/modeling-patterns/README.md`
- Create: `docs/02-data/beyond-relational/README.md`

**Interfaces:**
- Consumes: folder name `02-data` from Task 3.
- Produces: `modeling-patterns/idempotency-keys.md`, which Phase 5's `api-idempotency.md` (Task 7) cross-links to.

- [ ] **Step 1: Run the verification command and watch it fail**

Run:

```bash
find docs/02-data -name README.md 2>/dev/null | wc -l
```

Expected: `0`

- [ ] **Step 2: Create the parent `docs/02-data/README.md`**

Heading: `# Phase 2 — Data: Relational Depth First`
Subtitle: `The single highest-leverage area. Most backend performance problems are database problems.`

Then a `## Sections` list linking the seven subfolders, with each subsection's
root-README heading as link text:

```markdown
- [Relational fundamentals](relational-fundamentals/README.md)
- [Performance](performance/README.md)
- [Transactions and concurrency](transactions-and-concurrency/README.md)
- [ORMs and access layers](orms-and-access-layers/README.md)
- [Schema evolution](schema-evolution/README.md)
- [Modeling patterns](modeling-patterns/README.md)
- [Beyond relational](beyond-relational/README.md)
```

- [ ] **Step 3: Create `relational-fundamentals/README.md` and `performance/README.md`**

Each begins with `# ` plus the subsection name, then its bullets verbatim from
the root README, in order.

`relational-fundamentals/` — heading `# Relational fundamentals`, root README lines 68–72:

| Bullet | Target |
|---|---|
| Relational modeling: normalization to 3NF, and deliberate denormalization | `relational-modeling.md` |
| Keys and constraints: primary, foreign, unique, check, NOT NULL — pushing invariants to the DB | `keys-and-constraints.md` |
| Data types that matter: numeric vs float for money, `timestamptz`, JSONB, arrays, enums | `data-types.md` |
| SQL beyond CRUD: joins, aggregates, `GROUP BY`, window functions, CTEs, `LATERAL` | `sql-beyond-crud.md` |
| Upserts, `RETURNING`, bulk operations, `COPY`/batch insert | `upserts-and-bulk-operations.md` |

`performance/` — heading `# Performance`, root README lines 76–82:

| Bullet | Target |
|---|---|
| How B-tree indexes work; why an index is a sorted copy with a cost | `btree-indexes.md` |
| Composite index column order, partial indexes, covering indexes, index-only scans | `composite-and-partial-indexes.md` |
| Other index types: GIN, GiST, BRIN, hash — and when each applies | `index-types.md` |
| Reading `EXPLAIN ANALYZE`: seq scan vs index scan, nested loop vs hash vs merge join | `explain-analyze.md` |
| Query planner statistics, cardinality estimation, when the planner gets it wrong | `query-planner-statistics.md` |
| Slow query logs, `pg_stat_statements`, finding the real bottleneck | `finding-slow-queries.md` |
| Connection pooling: pool sizing math, PgBouncer, pool exhaustion symptoms | `connection-pooling.md` |

- [ ] **Step 4: Create `transactions-and-concurrency/README.md` and `orms-and-access-layers/README.md`**

`transactions-and-concurrency/` — heading `# Transactions and concurrency`, root README lines 86–92:

| Bullet | Target |
|---|---|
| ACID, concretely — what each letter buys you | `acid.md` |
| Isolation levels and their anomalies: dirty read, non-repeatable read, phantom, write skew | `isolation-levels.md` |
| MVCC in Postgres; bloat and `VACUUM` | `mvcc-and-vacuum.md` |
| Locking: row vs table, `SELECT FOR UPDATE`, lock ordering, deadlock detection | `locking-and-deadlocks.md` |
| Optimistic vs pessimistic concurrency control; version columns | `optimistic-vs-pessimistic-locking.md` |
| Transaction scope: where to begin/commit relative to your service layer | `transaction-scope.md` |
| Long transactions and why they're a production hazard | `long-transactions.md` |

`orms-and-access-layers/` — heading `# ORMs and access layers`, root README lines 96–101:

| Bullet | Target |
|---|---|
| What an ORM actually does: identity map, unit of work, change tracking, lazy loading | `what-an-orm-does.md` |
| The N+1 problem — detecting it, and eager loading strategies | `n-plus-one.md` |
| SQLAlchemy 2.0 (Core vs ORM), Prisma vs Drizzle vs Kysely vs TypeORM — tradeoffs | `orm-landscape.md` |
| The repository pattern: value, cost, and when it becomes an anti-pattern | `repository-pattern.md` |
| Dropping to raw SQL safely: parameterization, typed query builders | `raw-sql-safely.md` |
| Query result mapping and avoiding over-fetching | `result-mapping-and-overfetching.md` |

- [ ] **Step 5: Create `schema-evolution/README.md`, `modeling-patterns/README.md`, and `beyond-relational/README.md`**

`schema-evolution/` — heading `# Schema evolution`, root README lines 105–109:

| Bullet | Target |
|---|---|
| Migration tooling: Alembic, Prisma Migrate, Drizzle Kit | `migration-tooling.md` |
| Zero-downtime schema changes: expand → migrate → contract | `zero-downtime-migrations.md` |
| Dangerous migrations: locking DDL, adding NOT NULL, changing types, big backfills | `dangerous-migrations.md` |
| Backfill strategies: batched, resumable, throttled | `backfill-strategies.md` |
| Running migrations in CI/CD: ordering vs deploys, rollback strategy | `migrations-in-cicd.md` |

`modeling-patterns/` — heading `# Modeling patterns`, root README lines 113–117:

| Bullet | Target |
|---|---|
| Soft deletes: the case for and against | `soft-deletes.md` |
| Audit trails, history tables, temporal/bitemporal data | `audit-trails-and-temporal-data.md` |
| Multi-tenancy: shared schema vs schema-per-tenant vs DB-per-tenant, and row-level security | `multi-tenancy.md` |
| Idempotency keys and exactly-once request handling | `idempotency-keys.md` |
| Hierarchies, graphs, and recursive queries in SQL | `hierarchies-and-recursive-queries.md` |

`beyond-relational/` — heading `# Beyond relational`, root README lines 121–127:

| Bullet | Target |
|---|---|
| When a relational DB is the wrong tool (and how rarely that's true) | `when-not-relational.md` |
| Document stores (MongoDB/DynamoDB): access-pattern-first modeling, single-table design | `document-stores.md` |
| Key-value, wide-column, time-series, graph databases — the fit for each | `datastore-families.md` |
| Full-text search: Postgres `tsvector` vs Elasticsearch/OpenSearch tradeoffs | `full-text-search.md` |
| Vector search and embeddings storage (pgvector and alternatives) | `vector-search.md` |
| Object storage: S3 semantics, consistency, lifecycle, storage classes | `object-storage.md` |
| Polyglot persistence and the cost of every additional datastore you adopt | `polyglot-persistence.md` |

- [ ] **Step 6: Run the verification command again**

Run:

```bash
find docs/02-data -name README.md | wc -l
grep -c '^- \[' docs/02-data/*/README.md
```

Expected: `8`, then

```
docs/02-data/beyond-relational/README.md:7
docs/02-data/modeling-patterns/README.md:5
docs/02-data/orms-and-access-layers/README.md:6
docs/02-data/performance/README.md:7
docs/02-data/relational-fundamentals/README.md:5
docs/02-data/schema-evolution/README.md:5
docs/02-data/transactions-and-concurrency/README.md:7
```

Total across subsections: 42. The parent README's 7 section links are counted separately.

- [ ] **Step 7: Check link targets and commit**

Run:

```bash
grep -ohP '\]\(\K[^)]+' docs/02-data/*/README.md \
  | grep -vP '^[a-z0-9]+(-[a-z0-9]+)*\.md$' | grep -v 'CLAUDE\.md$' || echo CLEAN
git add docs/02-data
git commit -m "docs: add Phase 2 data topic indexes"
```

Expected: `CLEAN` before the commit output.

---

### Task 6: Phase 3 — Security

**Files:**
- Create: `docs/03-security/README.md`
- Create: `docs/03-security/foundations/README.md`
- Create: `docs/03-security/authentication/README.md`
- Create: `docs/03-security/authorization/README.md`
- Create: `docs/03-security/application-vulnerabilities/README.md`
- Create: `docs/03-security/data-protection-and-operations/README.md`

**Interfaces:**
- Consumes: folder name `03-security` from Task 3.
- Produces: `data-protection-and-operations/rate-limiting-as-security.md`, distinct from Phase 10's `rate-limiting-algorithms.md` (Task 9).

- [ ] **Step 1: Run the verification command and watch it fail**

Run:

```bash
find docs/03-security -name README.md 2>/dev/null | wc -l
```

Expected: `0`

- [ ] **Step 2: Create the parent `docs/03-security/README.md`**

Heading: `# Phase 3 — Security`
Subtitle: `Not a checklist to bolt on. A design constraint that shapes every layer.`

Then a `## Sections` list:

```markdown
- [Foundations](foundations/README.md)
- [Authentication](authentication/README.md)
- [Authorization](authorization/README.md)
- [Application vulnerabilities](application-vulnerabilities/README.md)
- [Data protection and operations](data-protection-and-operations/README.md)
```

- [ ] **Step 3: Create `foundations/README.md` and `authentication/README.md`**

`foundations/` — heading `# Foundations`, root README lines 137–139:

| Bullet | Target |
|---|---|
| Threat modeling with STRIDE; trust boundaries; thinking like an attacker | `threat-modeling.md` |
| Defense in depth, least privilege, fail-closed, secure defaults | `security-principles.md` |
| Authentication vs authorization vs accounting — precise definitions | `authn-authz-accounting.md` |

`authentication/` — heading `# Authentication`, root README lines 143–151:

| Bullet | Target |
|---|---|
| Password storage: Argon2id/bcrypt, salting, peppering, work factors, rehashing on login | `password-storage.md` |
| Session cookies vs JWTs: the real tradeoffs, and why "stateless" isn't free | `sessions-vs-jwts.md` |
| Cookie security: `HttpOnly`, `Secure`, `SameSite`, domain/path scoping | `cookie-security.md` |
| JWT correctly: signing algorithms, `alg` confusion, claims validation, key rotation via JWKS | `jwt-correctly.md` |
| Refresh tokens: rotation, reuse detection, revocation, token families | `refresh-tokens.md` |
| OAuth 2.1 and OIDC: authorization code + PKCE, client credentials, device flow | `oauth-and-oidc.md` |
| Identity providers: what you delegate to Auth0/Clerk/Cognito/Keycloak and what stays yours | `identity-providers.md` |
| MFA: TOTP, WebAuthn/passkeys, recovery codes | `mfa-and-passkeys.md` |
| Account lifecycle security: verification, password reset, enumeration resistance, session invalidation | `account-lifecycle-security.md` |

- [ ] **Step 4: Create `authorization/README.md` and `application-vulnerabilities/README.md`**

`authorization/` — heading `# Authorization`, root README lines 155–159:

| Bullet | Target |
|---|---|
| RBAC, ABAC, ReBAC (Google Zanzibar model) — expressiveness vs complexity | `authorization-models.md` |
| Enforcement points: middleware vs service layer vs database (RLS) | `enforcement-points.md` |
| IDOR / broken object-level authorization — the most common real-world API vuln | `broken-object-level-authorization.md` |
| Policy engines: OPA/Rego, Cedar, OpenFGA | `policy-engines.md` |
| Service-to-service auth: mTLS, signed JWTs, workload identity | `service-to-service-auth.md` |

`application-vulnerabilities/` — heading `# Application vulnerabilities`, root README lines 163–172:

| Bullet | Target |
|---|---|
| OWASP API Security Top 10, worked through with concrete examples | `owasp-api-top-10.md` |
| Injection: SQL, NoSQL, command, template, LDAP — parameterization as the fix | `injection.md` |
| SSRF: metadata endpoint attacks, allowlists, DNS rebinding | `ssrf.md` |
| CSRF: when it applies to APIs, double-submit vs SameSite | `csrf.md` |
| XSS from the backend's perspective: content types, sanitization boundaries, `nosniff` | `xss-backend-perspective.md` |
| Insecure deserialization: pickle, YAML, prototype pollution | `insecure-deserialization.md` |
| Mass assignment / over-posting; explicit allowlists | `mass-assignment.md` |
| Path traversal, ZIP slip, unrestricted file upload | `path-traversal-and-uploads.md` |
| Timing attacks and constant-time comparison | `timing-attacks.md` |
| CORS actually understood: preflight, credentials, why `*` fails, common misconfigurations | `cors.md` |

- [ ] **Step 5: Create `data-protection-and-operations/README.md`**

Heading `# Data protection and operations`, root README lines 176–185:

| Bullet | Target |
|---|---|
| Secrets management: Vault, cloud secret managers; never in env files in git | `secrets-management.md` |
| Encryption at rest vs in transit vs application-level; envelope encryption, KMS, key rotation | `encryption-and-key-management.md` |
| PII handling, data classification, tokenization, log redaction | `pii-and-data-classification.md` |
| Rate limiting and abuse prevention as security controls | `rate-limiting-as-security.md` |
| Supply chain: lockfiles, integrity hashes, SBOM, `npm audit`/`pip-audit`, Dependabot, typosquatting | `supply-chain-security.md` |
| Container and runtime hardening: non-root, read-only FS, minimal images | `container-hardening.md` |
| Security headers and HTTPS enforcement (HSTS, CSP as a backend concern) | `security-headers.md` |
| Audit logging: what to record, tamper-resistance, retention | `audit-logging.md` |
| Compliance orientation: GDPR/CCPA basics, SOC 2 concepts, data residency, right to erasure | `compliance-basics.md` |
| Responsible disclosure, CVE monitoring, patch cadence | `disclosure-and-patch-cadence.md` |

- [ ] **Step 6: Run the verification command again**

Run:

```bash
find docs/03-security -name README.md | wc -l
grep -c '^- \[' docs/03-security/*/README.md
```

Expected: `6`, then

```
docs/03-security/application-vulnerabilities/README.md:10
docs/03-security/authentication/README.md:9
docs/03-security/authorization/README.md:5
docs/03-security/data-protection-and-operations/README.md:10
docs/03-security/foundations/README.md:3
```

Total: 37.

- [ ] **Step 7: Check link targets and commit**

```bash
grep -ohP '\]\(\K[^)]+' docs/03-security/*/README.md \
  | grep -vP '^[a-z0-9]+(-[a-z0-9]+)*\.md$' | grep -v 'CLAUDE\.md$' || echo CLEAN
git add docs/03-security
git commit -m "docs: add Phase 3 security topic indexes"
```

Expected: `CLEAN` before the commit output.

---

### Task 7: Phase 4 and Phase 5 folder READMEs

**Files:**
- Create: `docs/04-caching-and-performance/README.md`
- Create: `docs/05-api-design/README.md`

**Interfaces:**
- Consumes: folder names from Task 3; `idempotency-keys.md` from Task 5.
- Produces: 20 + 20 topic filenames.

Phases 4 through 11 have no subtitle line in the root README. Use the phase's
description from `docs/README.md` (Task 3) as the subtitle instead, so each
folder README opens with a heading and one line of orientation.

- [ ] **Step 1: Run the verification command and watch it fail**

Run:

```bash
ls docs/04-caching-and-performance/README.md docs/05-api-design/README.md 2>&1 | tail -2
```

Expected: two "No such file or directory" errors.

- [ ] **Step 2: Create `docs/04-caching-and-performance/README.md`**

Heading `# Phase 4 — Caching and Performance`, root README lines 191–210:

| Bullet | Target |
|---|---|
| Cache theory: temporal/spatial locality, working set, hit ratio economics | `cache-theory.md` |
| The full cache hierarchy: browser → CDN → reverse proxy → app → DB buffer pool | `cache-hierarchy.md` |
| HTTP caching: `Cache-Control` directives, `ETag`, conditional requests, `Vary` | `http-caching.md` |
| `stale-while-revalidate` / `stale-if-error`; CDN cache keys and purging | `cdn-caching.md` |
| Redis fundamentals: data structures, TTLs, expiration, eviction policies (LRU/LFU/allkeys) | `redis-fundamentals.md` |
| Redis beyond caching: sorted sets, HyperLogLog, streams, Lua scripting, pipelines | `redis-beyond-caching.md` |
| Caching patterns: cache-aside, read-through, write-through, write-behind, refresh-ahead | `caching-patterns.md` |
| Cache key design, namespacing, and versioned keys for bulk invalidation | `cache-key-design.md` |
| Invalidation strategies: TTL, event-driven, write-invalidate — and why this is genuinely hard | `cache-invalidation.md` |
| Cache stampede / thundering herd: locks, singleflight, probabilistic early expiry, jittered TTL | `cache-stampede.md` |
| Negative caching, hot keys, cache penetration, sharding a cache | `cache-failure-patterns.md` |
| Consistency implications: stale reads, read-your-own-writes, acceptable staleness budgets | `cache-consistency.md` |
| In-process caching and its multi-instance pitfalls | `in-process-caching.md` |
| Precomputation: materialized views, rollup tables, denormalized read models | `precomputation.md` |
| Measuring first: profiling Python (`py-spy`, `cProfile`) and Node (`--prof`, clinic.js, flame graphs) | `profiling.md` |
| Load testing with k6 / Locust: open vs closed models, realistic workloads | `load-testing.md` |
| Latency statistics: why averages lie, p50/p95/p99, tail latency amplification | `latency-statistics.md` |
| Little's Law, queueing theory intuition, utilization vs latency curves | `queueing-theory.md` |
| Common wins: connection reuse, batching, N+1 elimination, payload size, compression | `common-performance-wins.md` |
| Knowing when _not_ to cache | `when-not-to-cache.md` |

- [ ] **Step 3: Create `docs/05-api-design/README.md`**

Heading `# Phase 5 — API Design and Contracts`, root README lines 216–235:

| Bullet | Target |
|---|---|
| Resource modeling, URL design, and Richardson maturity levels | `resource-modeling.md` |
| Pagination: offset vs keyset/cursor, stable ordering, total counts | `pagination.md` |
| Filtering, sorting, sparse fieldsets, expansion — without reinventing GraphQL badly | `filtering-and-field-selection.md` |
| Versioning strategies: URL, header, media type; deprecation policy and sunset headers | `api-versioning.md` |
| Idempotency for unsafe methods; idempotency keys end-to-end | `api-idempotency.md` |
| Bulk and batch endpoints; partial success semantics | `bulk-endpoints.md` |
| Long-running operations: 202 + status resource, polling vs callbacks | `long-running-operations.md` |
| OpenAPI-first design, codegen for clients and servers, contract testing (Pact) | `openapi-first-design.md` |
| Framework comparison: FastAPI, Litestar, Django REST vs Express, Fastify, NestJS, Hono | `framework-comparison.md` |
| GraphQL: schema design, resolvers, N+1 and DataLoader, pagination conventions | `graphql-basics.md` |
| GraphQL in production: depth/complexity limits, persisted queries, caching difficulties, federation | `graphql-in-production.md` |
| gRPC and Protobuf: streaming modes, schema evolution, when RPC beats REST | `grpc-and-protobuf.md` |
| tRPC and type-safe RPC in a TypeScript monorepo | `trpc-and-type-safe-rpc.md` |
| Realtime: polling vs long polling vs SSE vs WebSockets — tradeoffs and scaling each | `realtime-transports.md` |
| WebSocket production concerns: auth, heartbeats, reconnection, fan-out via pub/sub, backpressure | `websockets-in-production.md` |
| Webhooks you _send_: signing (HMAC), retries, ordering, replay protection, subscriber management | `sending-webhooks.md` |
| Webhooks you _receive_: verification, idempotency, fast-ack + async processing | `receiving-webhooks.md` |
| File handling: presigned URLs, multipart and resumable uploads, streaming, content validation | `file-uploads-and-downloads.md` |
| API gateways: routing, auth offload, rate limiting, request transformation | `api-gateways.md` |
| BFF pattern — where your frontend experience gives you an edge | `bff-pattern.md` |

- [ ] **Step 4: Add the cross-link note to the API idempotency entry**

At the end of `docs/05-api-design/README.md`, add:

```markdown
> `api-idempotency.md` covers the request-handling contract; the storage-side
> treatment lives in [Phase 2 — Idempotency keys](../02-data/modeling-patterns/idempotency-keys.md).
```

- [ ] **Step 5: Run the verification command again**

Run:

```bash
grep -c '^- \[' docs/04-caching-and-performance/README.md docs/05-api-design/README.md
```

Expected:

```
docs/04-caching-and-performance/README.md:20
docs/05-api-design/README.md:20
```

- [ ] **Step 6: Commit**

```bash
git add docs/04-caching-and-performance docs/05-api-design
git commit -m "docs: add Phase 4 and Phase 5 topic indexes"
```

---

### Task 8: Phase 6 and Phase 7 folder READMEs

**Files:**
- Create: `docs/06-async-and-distributed/README.md`
- Create: `docs/07-reliability-and-operations/README.md`

**Interfaces:**
- Consumes: folder names from Task 3.
- Produces: 26 + 18 topic filenames.

- [ ] **Step 1: Run the verification command and watch it fail**

Run:

```bash
ls docs/06-async-and-distributed/README.md docs/07-reliability-and-operations/README.md 2>&1 | tail -2
```

Expected: two "No such file or directory" errors.

- [ ] **Step 2: Create `docs/06-async-and-distributed/README.md`**

Heading `# Phase 6 — Asynchronous and Distributed Systems`, root README lines 241–266:

| Bullet | Target |
|---|---|
| Why async processing exists: latency decoupling, load smoothing, failure isolation | `why-async-processing.md` |
| Queues vs streams vs pub/sub — different primitives, different guarantees | `queues-streams-pubsub.md` |
| Broker landscape: Redis Streams, RabbitMQ, SQS/SNS, Kafka, NATS — selection criteria | `broker-landscape.md` |
| Task queues in practice: Celery, RQ, arq, Dramatiq; BullMQ, Graphile Worker | `task-queue-libraries.md` |
| Durable execution engines: Temporal, Inngest, Restate — what problem they actually solve | `durable-execution.md` |
| Delivery semantics: at-most-once, at-least-once, "exactly-once" and why it's a half-truth | `delivery-semantics.md` |
| Idempotent consumers and deduplication strategies | `idempotent-consumers.md` |
| Ordering guarantees, partitioning keys, consumer groups, rebalancing | `ordering-and-partitioning.md` |
| Retries: exponential backoff with jitter, retry budgets, avoiding retry storms | `retries-and-backoff.md` |
| Dead letter queues, poison messages, replay tooling | `dead-letter-queues.md` |
| Scheduled and recurring work: cron correctness, missed runs, timezone traps | `scheduled-jobs.md` |
| Distributed locks: Redlock, why it's contested, fencing tokens, lease-based approaches | `distributed-locks.md` |
| Leader election and singleton workers | `leader-election.md` |
| The dual-write problem and the transactional outbox pattern | `transactional-outbox.md` |
| Change Data Capture (Debezium) as an integration primitive | `change-data-capture.md` |
| Event-driven architecture: event notification vs event-carried state transfer | `event-driven-architecture.md` |
| Event schema design, versioning, and the schema registry | `event-schema-versioning.md` |
| CQRS: separating read and write models; when the complexity is justified | `cqrs.md` |
| Event sourcing: append-only logs, projections, replay, snapshots — and the real costs | `event-sourcing.md` |
| Sagas and compensating transactions; orchestration vs choreography | `sagas-and-compensation.md` |
| Two-phase commit and why distributed transactions are usually avoided | `two-phase-commit.md` |
| Consistency models: strong, eventual, causal, read-your-writes, monotonic reads | `consistency-models.md` |
| CAP and PACELC — stated precisely, not as a slogan | `cap-and-pacelc.md` |
| Time in distributed systems: clock skew, NTP, logical clocks, vector clocks | `clocks-in-distributed-systems.md` |
| Consensus intuition: Raft, quorums, split brain | `consensus-and-quorums.md` |
| Service communication: sync vs async, and the latency/coupling tradeoff | `sync-vs-async-communication.md` |

- [ ] **Step 3: Create `docs/07-reliability-and-operations/README.md`**

Heading `# Phase 7 — Reliability and Operations`, root README lines 272–289:

| Bullet | Target |
|---|---|
| Failure taxonomy: crash, partial, gray failures, cascading failure | `failure-taxonomy.md` |
| Timeouts everywhere; deadline propagation and request budgets | `timeouts-and-deadlines.md` |
| Circuit breakers, bulkheads, load shedding, admission control | `resilience-patterns.md` |
| Backpressure: propagating it correctly instead of buffering to death | `backpressure.md` |
| Graceful shutdown: SIGTERM handling, connection draining, in-flight work | `graceful-shutdown.md` |
| Health checks: liveness vs readiness vs startup; the danger of deep health checks | `health-checks.md` |
| Degraded modes and fallbacks; designing what "partially working" looks like | `degraded-modes.md` |
| Observability pillars: logs, metrics, traces, and how they compose | `observability-pillars.md` |
| Structured logging, correlation/request IDs, log levels, sampling, cost control | `logging-in-production.md` |
| Metrics: counters/gauges/histograms, cardinality explosions, RED and USE methods | `metrics.md` |
| Distributed tracing and OpenTelemetry: spans, context propagation, instrumenting Python and Node | `distributed-tracing.md` |
| Dashboards that answer questions; alerting on symptoms, not causes | `dashboards-and-alerting.md` |
| SLIs, SLOs, error budgets — and using them to make engineering decisions | `slos-and-error-budgets.md` |
| On-call practice: runbooks, escalation, incident command, blameless postmortems | `on-call-and-incidents.md` |
| Backups: full vs incremental, point-in-time recovery, and testing restores | `backups.md` |
| Disaster recovery: RPO/RTO, failover strategy, multi-AZ vs multi-region | `disaster-recovery.md` |
| Chaos engineering and game days | `chaos-engineering.md` |
| Capacity planning and forecasting | `capacity-planning.md` |

- [ ] **Step 4: Run the verification command again**

Run:

```bash
grep -c '^- \[' docs/06-async-and-distributed/README.md docs/07-reliability-and-operations/README.md
```

Expected:

```
docs/06-async-and-distributed/README.md:26
docs/07-reliability-and-operations/README.md:18
```

- [ ] **Step 5: Commit**

```bash
git add docs/06-async-and-distributed docs/07-reliability-and-operations
git commit -m "docs: add Phase 6 and Phase 7 topic indexes"
```

---

### Task 9: Phases 8 through 11 folder READMEs

**Files:**
- Create: `docs/08-infrastructure-and-delivery/README.md`
- Create: `docs/09-testing-and-quality/README.md`
- Create: `docs/10-scaling/README.md`
- Create: `docs/11-advanced/README.md`

**Interfaces:**
- Consumes: folder names from Task 3.
- Produces: 16 + 15 + 17 + 16 topic filenames.

- [ ] **Step 1: Run the verification command and watch it fail**

Run:

```bash
ls docs/08-infrastructure-and-delivery/README.md docs/09-testing-and-quality/README.md \
   docs/10-scaling/README.md docs/11-advanced/README.md 2>&1 | tail -4
```

Expected: four "No such file or directory" errors.

- [ ] **Step 2: Create `docs/08-infrastructure-and-delivery/README.md`**

Heading `# Phase 8 — Infrastructure and Delivery`, root README lines 295–310:

| Bullet | Target |
|---|---|
| Containers: images, layers, caching, multi-stage builds for Python and Node | `container-images.md` |
| Image hygiene: slim/distroless bases, non-root users, `.dockerignore`, reproducible builds | `image-hygiene.md` |
| Docker Compose for local development that mirrors production | `docker-compose-local-dev.md` |
| Orchestration concepts: pods, deployments, services, ingress, configmaps, secrets | `kubernetes-concepts.md` |
| Resource requests/limits, autoscaling (HPA), OOM kills, CPU throttling | `resource-limits-and-autoscaling.md` |
| Simpler alternatives: ECS/Fargate, Fly.io, Render, Railway — and when they're the right call | `managed-platforms.md` |
| Serverless: cold starts, execution limits, connection pooling problems, cost model | `serverless.md` |
| Load balancers L4 vs L7, reverse proxies (Nginx, Caddy, Envoy), service mesh overview | `load-balancers-and-proxies.md` |
| Networking: VPCs, subnets, security groups, private links, egress control | `cloud-networking.md` |
| DNS operationally: TTLs, failover, blue/green cutover | `dns-operations.md` |
| Infrastructure as Code: Terraform or Pulumi, state management, drift | `infrastructure-as-code.md` |
| CI/CD pipelines: build once, promote artifacts, environment parity | `cicd-pipelines.md` |
| Deployment strategies: rolling, blue/green, canary, progressive delivery with flags | `deployment-strategies.md` |
| Ephemeral preview environments and seeded test data | `preview-environments.md` |
| Secrets in CI, OIDC-based cloud auth, least-privilege pipelines | `ci-secrets-and-oidc.md` |
| Cost engineering: egress charges, storage tiers, right-sizing, the cost of idle capacity | `cost-engineering.md` |

- [ ] **Step 3: Create `docs/09-testing-and-quality/README.md`**

Heading `# Phase 9 — Testing and Quality`, root README lines 316–330:

| Bullet | Target |
|---|---|
| Test strategy: pyramid vs trophy; what each level is actually for | `test-strategy.md` |
| Test doubles: stub, mock, fake, spy — and how DI makes them unnecessary as often as possible | `test-doubles.md` |
| Unit testing business logic with no I/O; designing for that | `unit-testing-business-logic.md` |
| Integration testing against real dependencies with Testcontainers | `integration-testing.md` |
| Database test strategy: transactional rollback, truncation, migrations in tests, factories | `database-testing.md` |
| Deterministic tests: freezing time, seeded randomness, controlling concurrency | `deterministic-tests.md` |
| Testing async code and background workers | `testing-async-code.md` |
| Contract testing between services; consumer-driven contracts | `contract-testing.md` |
| Property-based testing: Hypothesis (Python), fast-check (TS) | `property-based-testing.md` |
| Snapshot and approval testing — the good and the trap | `snapshot-testing.md` |
| Mutation testing to measure suite quality | `mutation-testing.md` |
| Load, soak, spike, and stress tests as part of the pipeline | `performance-tests-in-ci.md` |
| Static analysis: mypy/pyright strict, TypeScript strict mode, ruff, ESLint | `static-analysis.md` |
| Making illegal states unrepresentable with the type system | `types-for-correctness.md` |
| Test flakiness: causes, quarantine, and fixing rather than retrying | `test-flakiness.md` |

- [ ] **Step 4: Create `docs/10-scaling/README.md`**

Heading `# Phase 10 — Scaling`, root README lines 336–352:

| Bullet | Target |
|---|---|
| Vertical vs horizontal scaling; knowing which problem you have | `vertical-vs-horizontal-scaling.md` |
| Statelessness as an enabler; where state actually lives | `statelessness.md` |
| Session storage and sticky sessions — and why to avoid them | `session-storage.md` |
| Read replicas: replication lag, read-your-writes, routing reads safely | `read-replicas.md` |
| Failover, promotion, synchronous vs asynchronous replication | `replication-and-failover.md` |
| Partitioning within a database (Postgres declarative partitioning) | `table-partitioning.md` |
| Sharding: key selection, hash vs range, consistent hashing, resharding pain | `sharding.md` |
| Cross-shard queries, distributed joins, global uniqueness under sharding | `cross-shard-queries.md` |
| Distributed SQL (CockroachDB, Vitess, Citus, Aurora) — what they buy you | `distributed-sql.md` |
| Rate limiting algorithms: fixed/sliding window, token bucket, leaky bucket | `rate-limiting-algorithms.md` |
| Distributed rate limiting and quota enforcement without a bottleneck | `distributed-rate-limiting.md` |
| Fan-out strategies: write-time vs read-time (the timeline problem) | `fan-out-strategies.md` |
| Multi-region: latency, data residency, active-active vs active-passive, conflict resolution | `multi-region.md` |
| Edge compute and CDN-level logic | `edge-compute.md` |
| Noisy neighbors, per-tenant isolation, fairness and prioritization | `tenant-isolation-and-fairness.md` |
| The monolith → services decision: coupling, team topology, transaction boundaries | `monolith-to-services.md` |
| Strangler fig migrations and extracting a service safely | `strangler-fig-migration.md` |

- [ ] **Step 5: Create `docs/11-advanced/README.md`**

Heading `# Phase 11 — Advanced and Specialized`, root README lines 358–373:

| Bullet | Target |
|---|---|
| Python performance: `asyncio` internals, uvloop, structured concurrency, TaskGroups | `python-async-internals.md` |
| Python at the edges: C extensions, Cython, Rust via PyO3, free-threaded builds | `python-native-extensions.md` |
| Python memory: reference counting, GC, leak hunting, `tracemalloc` | `python-memory.md` |
| Node internals: event loop phases in detail, streams and backpressure, worker threads | `node-internals.md` |
| Node memory: V8 heap, GC pauses, heap snapshots, leak patterns | `node-memory.md` |
| Database internals: storage layout, WAL, buffer pool, B-tree vs LSM tree, compaction | `database-internals.md` |
| Writing middleware, an ASGI app, or a small HTTP server from scratch | `building-from-scratch.md` |
| Designing a protocol: framing, versioning, backwards/forwards compatibility | `protocol-design.md` |
| OLTP vs OLAP; data warehouses, ELT, dbt, analytics without wrecking production | `oltp-vs-olap.md` |
| Streaming data processing and materialized views over event streams | `stream-processing.md` |
| Serving ML/LLM workloads: batching, GPU vs CPU constraints, queues, timeouts, streaming responses | `serving-ml-workloads.md` |
| Retrieval infrastructure: vector stores, hybrid search, embedding pipelines | `retrieval-infrastructure.md` |
| Platform engineering: golden paths, internal libraries, developer experience | `platform-engineering.md` |
| Buy vs build, and evaluating vendors and open-source dependencies | `buy-vs-build.md` |
| Writing RFCs and design docs; running a design review | `rfcs-and-design-docs.md` |
| System design practice: sketching, estimating, and defending tradeoffs out loud | `system-design-practice.md` |

- [ ] **Step 6: Run the verification command again**

Run:

```bash
grep -c '^- \[' docs/08-infrastructure-and-delivery/README.md docs/09-testing-and-quality/README.md \
  docs/10-scaling/README.md docs/11-advanced/README.md
```

Expected:

```
docs/08-infrastructure-and-delivery/README.md:16
docs/09-testing-and-quality/README.md:15
docs/10-scaling/README.md:17
docs/11-advanced/README.md:16
```

- [ ] **Step 7: Commit**

```bash
git add docs/08-infrastructure-and-delivery docs/09-testing-and-quality docs/10-scaling docs/11-advanced
git commit -m "docs: add Phase 8 through Phase 11 topic indexes"
```

---

### Task 10: Capstones folder README

**Files:**
- Create: `docs/capstones/README.md`

**Interfaces:**
- Consumes: folder name `capstones` from Task 3.
- Produces: 6 project note filenames.

Capstone entries are numbered projects, not checkbox bullets, so this README uses
a numbered list and keeps each project's phase attribution from the root README.

- [ ] **Step 1: Run the verification command and watch it fail**

Run:

```bash
test -f docs/capstones/README.md && echo PRESENT || echo ABSENT
```

Expected: `ABSENT`

- [ ] **Step 2: Create `docs/capstones/README.md` with exactly this content**

```markdown
# Capstone Projects

Build these in order. Each one forces the concepts from its phases into your hands.

Project **code lives in a separate repository per project**. The files linked here
hold notes and design decisions only — what was decided, what went wrong, what the
measured result was.

1. [URL shortener with real constraints](url-shortener.md) — ID generation, caching, rate limiting, analytics via async writes. _(Phases 0, 2, 4)_
2. [Multi-tenant task API](multi-tenant-task-api.md) — layered architecture with DI, RBAC, RLS, cursor pagination, OpenAPI, full test suite. _(Phases 1, 3, 5, 9)_
3. [Payments-style ledger](payments-ledger.md) — double-entry accounting, transactional integrity, idempotency keys, exactly-once webhooks, reconciliation. _(Phases 2, 5, 6)_
4. [Event-driven order pipeline](event-driven-order-pipeline.md) — outbox pattern, message broker, saga with compensation, DLQ, replay tooling. _(Phase 6)_
5. [High-throughput read service](high-throughput-read-service.md) — multi-layer caching, stampede protection, load tested to a documented p99 target with tracing and dashboards. _(Phases 4, 7)_
6. [Ship one of the above properly](ship-it-properly.md) — containerized, IaC-provisioned, CI/CD with zero-downtime migrations, SLOs, alerts, and a tested restore. _(Phases 7, 8)_
```

- [ ] **Step 3: Run the verification command again**

Run:

```bash
grep -c '^[0-9]\. \[' docs/capstones/README.md
```

Expected: `6`

- [ ] **Step 4: Commit**

```bash
git add docs/capstones/README.md
git commit -m "docs: add capstone project index"
```

---

### Task 11: Strip checkboxes from the root README, then verify the whole scaffold

**Files:**
- Modify: `README.md` (checkbox prefixes only)

**Interfaces:**
- Consumes: every folder README from Tasks 4–10.
- Produces: nothing. This is the final task.

`status:` frontmatter is the single source of truth for progress, so the root
README's checkboxes have to go — two trackers drift. Only the `- [ ] ` prefix is
removed; every other character of the file stays as it is.

- [ ] **Step 1: Count the checkboxes before touching anything**

Run:

```bash
grep -c '^- \[ \] ' README.md
```

Expected: `265`. Record the number you actually see — Step 4 compares against it.

Per-phase breakdown, for locating a discrepancy: Phase 0 = 17, Phase 1 = 21,
Phase 2 = 42, Phase 3 = 37, Phase 4 = 20, Phase 5 = 20, Phase 6 = 26, Phase 7 = 18,
Phase 8 = 16, Phase 9 = 15, Phase 10 = 17, Phase 11 = 16.

- [ ] **Step 2: Strip the checkbox prefixes**

Run:

```bash
sed -i 's/^- \[ \] /- /' README.md
```

This matches only lines beginning with `- [ ] `. The capstone list uses `1. **…**`
and the tutor-prompt section uses numbered items, so neither is touched.

- [ ] **Step 3: Verify no checkboxes remain**

Run:

```bash
grep -c '^- \[ \] ' README.md || echo ZERO
```

Expected: `ZERO` (grep exits non-zero when it finds no matches).

- [ ] **Step 4: Verify no bullet was lost or reworded**

Run:

```bash
grep -c '^- ' README.md
git diff --stat README.md
git diff README.md | grep -c '^[-+]' 
```

Expected: the bullet count from Step 1 (plus any bullets that were already
checkbox-free), and a diff whose changed lines are exactly twice the Step 1 count
plus the two `+++`/`---` header lines. If `git diff` shows any change other than a
leading `- [ ] ` becoming `- `, revert with `git checkout README.md` and redo Step 2.

- [ ] **Step 5: Verify every phase and subsection has a folder**

Run:

```bash
ls -d docs/*/ | wc -l
find docs -mindepth 2 -maxdepth 2 -type d -not -path 'docs/superpowers/*' | wc -l
```

Expected: `14` (twelve phases, `capstones`, `superpowers`), then `12` (seven Phase 2
subsections plus five Phase 3 subsections).

- [ ] **Step 6: Verify the total topic count matches the curriculum**

Run:

```bash
grep -rhc '^- \[' docs --include=README.md | paste -sd+ | bc
```

Expected: `292`, composed of:

- 265 topic links across the phase and subsection READMEs
- 12 section links in the Phase 2 (7) and Phase 3 (5) parent READMEs
- 15 links in `docs/README.md` — 12 phases, 1 capstone, 2 meta

`docs/capstones/README.md` contributes 0 because its entries are a numbered list.
If your Step 1 count differed from 265, expect that number plus 27 instead. A
mismatch means a bullet was dropped or duplicated; find it by diffing the bullet
text of the root README against the folder READMEs.

- [ ] **Step 7: Verify every topic link matches the filename convention**

Run:

```bash
find docs -name README.md -not -path 'docs/superpowers/*' -exec grep -ohP '\]\(\K[^)]+' {} + \
  | grep -vP '^[a-z0-9]+(-[a-z0-9]+)*\.md$' \
  | grep -vP '^([a-z0-9-]+/README\.md|superpowers/(specs|plans)/|\.\./.*)$' || echo CLEAN
```

Expected: `CLEAN`. The second `grep -v` allows three legitimate shapes: subfolder
`README.md` links, the two `superpowers/` directory links in `docs/README.md`, and
`../` relative links to `CLAUDE.md`, the root README, and cross-phase references.

- [ ] **Step 8: Verify no topic files were created by mistake**

Run:

```bash
find docs -name '*.md' -not -name README.md -not -path 'docs/superpowers/*' || echo NONE
```

Expected: `NONE`. Topic files are created when a topic is studied, not now.

- [ ] **Step 9: Verify no toolchain files were added**

Run:

```bash
find . -maxdepth 2 -name 'package.json' -o -maxdepth 2 -name 'pyproject.toml' \
  -o -maxdepth 2 -name 'requirements.txt' -not -path './.git/*' | grep . || echo CLEAN
```

Expected: `CLEAN`

- [ ] **Step 10: Commit**

```bash
git add README.md
git commit -m "docs: remove README checkboxes in favor of frontmatter status"
git log --oneline
```

Expected: sixteen commits on top of `e62d7bf initial commit`, composed of:

- 11 task commits, one per task, this task's included
- 2 documentation commits — the spec and this plan
- 3 controller correction commits: `e5a2987` (four pre-flight plan defects), `171d3aa`
  (two miscounted verification expectations), and one amending this very expectation

Fifteen of those exist before this task; this task's commit is the sixteenth.

---

## Self-Review

**Spec coverage:**

| Spec section | Task |
|---|---|
| `.gitignore` | 1 |
| `CLAUDE.md` contents (all 5 numbered items) | 2 |
| `docs/README.md` index | 3 |
| Phase folders `00`–`11` | 4, 5, 6, 7, 8, 9 |
| Phase 2 seven subsection subfolders | 5 |
| Phase 3 five subsection subfolders | 6 |
| `capstones/` | 10 |
| README in every folder and subfolder | 4–10 |
| Filename convention | Global Constraints; enforced in 4, 5, 6, 11 |
| Topic template | 2 |
| Status vocabulary | 2 |
| Root README checkbox removal | 11 |
| Verification check 1 (every phase/subsection has a folder) | 11 Step 5 |
| Verification check 2 (every bullet in exactly one folder README) | 11 Step 6 |
| Verification check 3 (every link matches the slug convention) | 11 Step 7 |

No gaps.

**Placeholder scan:** No TBD, TODO, "similar to Task N", or "add appropriate
handling". Every file's content is either given verbatim or specified as a
heading plus a complete bullet-to-slug table.

**Name consistency:** Folder names in Task 3's `docs/README.md` match the paths
used in Tasks 4–10 exactly. `idempotency-keys.md` (Task 5) and
`api-idempotency.md` (Task 7) are the two idempotency files, cross-linked in Task
7 Step 4 and named identically in both places. `rate-limiting-as-security.md`
(Task 6) and `rate-limiting-algorithms.md` plus `distributed-rate-limiting.md`
(Task 9) are the three rate-limiting files, all distinct.
