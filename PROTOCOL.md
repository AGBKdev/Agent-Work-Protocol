# Agent Work Protocol

Drop this folder into any project. It defines how an AI agent (Claude,
Claude Code, or any capable assistant) works on that project.

## The three files

1. **task.json** - Source of truth. Structured claims with sources. QA
   verifies this file only.
2. **MEMORY.md** - Human-readable state for session pickup and agent
   handoff. Commentary, not authority.
3. **PROTOCOL.md** - This file. The rules.

## Rules

### 1. Docs first

Before asserting anything consequential (API signatures, config syntax,
version behavior, licensing terms), fetch the current official source.
Approved source classes, in order of preference:

- Official documentation for the pinned version in `environment`
- Official release notes / changelogs
- License text itself (for licensing questions)
- Vendor knowledge bases
- Direct code inspection

Reddit, Stack Overflow, YouTube, blog posts: allowed for *leads* only.
Any fact sourced from them must be re-verified against an approved source
before it enters task.json as verified.

Anything stated from model memory is tagged `source_type:
"unverified_memory"` and `verified: false` until checked.

### 2. Claims are atomic

One checkable statement per claim. Not "the render manager works with our
setup" but "Flamenco 3.9.2 supports two-way path variables for
cross-platform output paths."

### 3. QA is a separate pass

QA reads ONLY task.json. For each claim: fetch the source, confirm the doc
actually supports the statement for the pinned version. Mark verified
true/false and record failures in the `qa` block. QA does not read the
chat transcript. In Claude Code, run QA as a subagent via the Task tool
with task.json as its only input.

### 4. JSON is the interface

Agents communicate through task.json, not through prose summaries of each
other. This prevents compounding hallucination.

### 5. Session hygiene

End of every session: update MEMORY.md (state, next, session log) and commit
locally. **Verify the repo root before staging** — `git add -A` walks up the
tree and stages whatever repo it finds, which may not be this project:

```bash
git rev-parse --show-toplevel        # MUST print the project root
git add -A && git commit -m "session: <summary>"
```

If that first command prints your home directory, stop. A stray repo exists
above the project and `git add -A` would stage `.ssh/`, `.aws/` and shell
history. Fix the root before committing anything.

### 6. Findings are promoted, not archived

At the end of every session, ask of each finding: **would this hold on a
different project, with different assets?**

- **Yes** -> it goes into your knowledge base NOW, not "later":
  `KNOWLEDGE/LAWS.md` (in this folder's layout; put yours wherever your
  projects can all reach it). One crisp sentence, a pointer to where the
  evidence is, and a confidence tag: `measured` (a number, under controlled
  conditions), `single observation` (seen once and generalised), or
  `reasoned, not measured` (inferred, never tested - a hypothesis, not a rule).
- **No** -> it stays in this project's MEMORY.md as a project fact.

Give every law an **id**, not a position: `YYYY-MM-DD-short-kebab-slug`, where the
date is when it was recorded and the slug comes from the law's first clause, e.g.
`2026-07-31-conditional-never-evaluated-true-test`. Reference laws by id
everywhere — in the index, in project notes, in commit messages.

Why ids and not numbers: two sessions once promoted findings within the
same hour and BOTH claimed law 88, because a sequential position is allocated from
the current length of a file that another process is also appending to. Ids need
no coordination. If two sessions ever mint the same id they have found the same
law, which is a merge, not a collision.

When new evidence contradicts a law, do NOT delete it. Mark it *superseded*
with the evidence that killed it. A retired law is as useful as a live one.

Why this is a rule rather than a suggestion: 86 transferable laws had to be
recovered out of a 533-line project narrative by a dedicated audit, because
nobody promoted them as they were found. Every one was already known and
already paid for. That is the cost of skipping this step.

## Setup (once per project)

```bash
cp -r agent-work-protocol /path/to/project/_agent
cd /path/to/project

# Init ONLY if not already inside a repo. Never run a bare `git init` from an
# unverified cwd — that is exactly how a repo ends up rooted at $HOME.
git rev-parse --show-toplevel 2>/dev/null || git init -b main
git rev-parse --show-toplevel        # verify: must be THIS project

# A .gitignore must exist before the first `git add -A`.
git add _agent .gitignore && git commit -m "add agent protocol"
```

Optional backup to a NAS or second disk:

```bash
rsync -av --delete /path/to/project/_agent/ \
  /path/to/backups/agent-state/PROJECT_NAME/
```
