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

### Adding a topic

When the curriculum gains a bullet:

1. Add the bullet to the root `README.md`, under the phase it belongs to.
2. Add one link for it in exactly one folder README — the phase folder, or the
   subsection folder for Phases 2 and 3. The link text is the bullet verbatim,
   minus nothing and reworded nowhere.
3. Choose a slug that does not already exist anywhere under `docs/`. Slugs are
   globally unique, not unique per folder.

Two bullets covering adjacent ground get two files with distinct slugs that link to
each other. They are never merged. One bullet, one file, one folder — that
correspondence is what makes this tree trustworthy, and it is only as good as the
last topic added.

## Topic template

Every topic doc under a phase folder uses this frontmatter and these sections, in this order:

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

`phase:` is the numbered phase folder — `02-data`, not `performance`. A topic inside a
Phase 2 or Phase 3 subsection adds a second field naming that subfolder:

```
phase: 02-data
section: performance
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

### Capstone notes

`docs/capstones/` holds notes on builds, not topics, so it uses a different shape.
Frontmatter is `phase: capstones` with the same `status:` and `updated:` fields, and
the sections are:

```
## What I built
## Design decisions and why
## What went wrong
## Measured result
## Sources
```

The project code itself lives in its own repository. What belongs here is the
reasoning and the evidence — the decisions, the failures, and the numbers.

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
- **Plant the bug rather than explaining the pitfall.** Show a realistic
  implementation with a subtle fault in it and let it be found before naming it.
  Recognising a correct explanation is not the same skill as spotting a wrong
  implementation.
- **Ask "what would change your answer?"** after any recommendation, so the
  conditions it depends on stay visible.
- **Quiz with scenarios, not definitions,** and do not reveal the answer until
  an answer has been committed to.
- **Say "I don't know."** A gap named is worth more than a confident paragraph.

## Conventions

- Prose in docs is plain and specific. No filler, no marketing register.
- Cite sources with links in the **Sources** section; prefer primary docs and
  papers over blog posts.
- `scratch/` is gitignored — put throwaway experiments there.
- Never add a language toolchain to this repository.
