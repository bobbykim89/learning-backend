# Repository Scaffold — Design

**Date:** 2026-08-16
**Status:** Approved

## Purpose

`learn-backend` is a personal study repository for AI-assisted learning of backend
engineering. The root `README.md` already holds the curriculum: twelve phases
(Phase 0–11) plus six capstone projects, 265 topic bullets, each scoped to
one 30–90 minute study session.

This design defines the repository scaffold — folder structure, `.gitignore`, and
`CLAUDE.md` — that all future documentation will inherit. It does not write any
topic documentation.

## Scope

In scope:

- `.gitignore`
- `CLAUDE.md`
- `docs/README.md` index
- Phase folders, subsection subfolders, and a `README.md` in each
- Removing the checkboxes from the root `README.md`

Out of scope:

- Topic documentation content (created per study session, later)
- Capstone project code (lives in separate repositories)
- Any script, toolchain, or CI configuration

## Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Doc granularity | One file per topic bullet | A whole topic fits in an AI context window; git history shows what was studied when |
| Code in repo | Docs only, snippets inline in fenced blocks | No toolchain to maintain; capstones live elsewhere |
| Topic structure | Fixed template mirroring the root README's tutor ladder | Required sections make a shallow doc visibly incomplete |
| Scaffold depth | Folders and folder READMEs now; topic files created when studied | Navigable tree without 265 empty files diluting search and history |
| Progress tracking | Frontmatter `status:` is canonical | One place to edit; greppable; no drift between doc and dashboard |
| Tree depth | Nest only where the root README nests | The curriculum is already the taxonomy; inventing groups would drift from it |

## Structure

Phase folders carry an `NN-` prefix so they sort in curriculum order. `capstones/`
has no prefix because it is not a phase. Phase 2 and Phase 3 have subsection
subfolders mirroring their `###` headings; the other ten phases are flat.

```
.gitignore
CLAUDE.md
README.md                       (curriculum; checkboxes removed)
docs/
  README.md                     (index of phases)
  00-mental-models/
    README.md
  01-architecture/
    README.md
  02-data/
    README.md
    relational-fundamentals/
    performance/
    transactions-and-concurrency/
    orms-and-access-layers/
    schema-evolution/
    modeling-patterns/
    beyond-relational/
  03-security/
    README.md
    foundations/
    authentication/
    authorization/
    application-vulnerabilities/
    data-protection-and-operations/
  04-caching-and-performance/
  05-api-design/
  06-async-and-distributed/
  07-reliability-and-operations/
  08-infrastructure-and-delivery/
  09-testing-and-quality/
  10-scaling/
  11-advanced/
  capstones/
  superpowers/specs/            (meta: design docs like this one; not curriculum)
```

Every folder and subsection subfolder contains a `README.md` listing its topics as
markdown links to their planned filenames. Those files do not exist until the topic
is studied; the links are the naming contract.

### Filename convention

Kebab-case slug derived from the topic bullet, trimmed to the core concept, no
number prefix, target five words or fewer:

| Bullet | Filename |
|---|---|
| How B-tree indexes work; why an index is a sorted copy with a cost | `btree-indexes.md` |
| Composite index column order, partial indexes, covering indexes, index-only scans | `composite-and-specialized-indexes.md` |
| CAP and PACELC — stated precisely, not as a slogan | `cap-and-pacelc.md` |

Each bullet maps to exactly one file, and each file lives in exactly one folder.

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

### Status vocabulary

- `not-started` — file exists as a stub only
- `learning` — being read and written up
- `practiced` — something was built with the concept

Per the root README's own rule, `practiced` requires having built something, not
having read about it.

## CLAUDE.md contents

A single file, read by the AI at the start of every session, containing:

1. Repository purpose and the fact that it is documentation only
2. The tree map and the filename convention
3. The topic template verbatim, and the status vocabulary
4. Anti-fluency rules, derived from the root README's tutor section:
   - Argue both sides of a tradeoff before recommending
   - Ask for failure modes and operational cost before endorsing any tool
   - Flag version-specific claims as unverified against primary docs
   - Prefer scenario questions over definitions when quizzing
5. The instruction to update `status` and `updated` at the end of a session

## .gitignore

Docs-only repository, so nothing language-specific:

```
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

`scratch/` is the documented location for throwaway local experiments.
`.superpowers/` holds the execution ledger and review artifacts, which are scratch
state rather than repository content.

## Verification

No tests apply to a documentation scaffold. Three mechanical checks, confirmed by
inspection before reporting completion:

1. Every phase heading and every `###` subsection in the root README has a
   corresponding folder.
2. Every topic bullet in the root README appears in exactly one folder README.
3. Every link in every folder README points at a filename matching the slug
   convention.
