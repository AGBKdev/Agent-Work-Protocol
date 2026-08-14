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
"unverified_memory"` and `confidence: "unverified"` until checked.

### 2. Claims are atomic

One checkable statement per claim. Not "the render manager works with our
setup" but "Flamenco 3.9.2 supports two-way path variables for
cross-platform output paths."

### 3. QA is a separate pass

QA reads task.json and the artifacts task.json points at — never the chat
transcript. Run it as a subagent whose only input is the file.

**Two kinds of claim, two verbs**, selected by the claim's `method`. A
`documented` claim is checked by fetching its `source` and confirming the doc
supports the statement *for the pinned version*. An `empirical` claim (or one
whose `source_type` is `code_inspection`) cannot be checked that way — there is
no document. It is checked by re-running the command in the claim's
`reproduction` field and comparing against the expected value.

That makes `reproduction` mandatory for empirical claims. A measurement nobody
recorded how to repeat is not a claim, it is a memory: mark it
`NOT_INDEPENDENTLY_CHECKABLE` and say what is missing, rather than passing it.

**One state field.** A claim's status lives in `confidence`, and nowhere else.
QA *returns* a verdict; the session agent appends it to `qa.verifications` and
derives `confidence` from the latest one:

| verdict | confidence |
|---|---|
| CONFIRMED / REPRODUCED | `verified` |
| PARTIALLY_SUPPORTED | `partially_supported` |
| DOES_NOT_SUPPORT / NOT_REPRODUCED / REFUTED | `refuted` |
| SOURCE_UNREACHABLE | `unreachable` |
| NOT_INDEPENDENTLY_CHECKABLE | `unverified` |
| PIN_MOVED | `stale` |

Every confidence value is reachable from a verdict, and only from a verdict.
That is what makes the log the explanation: if a claim is `stale`, there is a
`PIN_MOVED` entry saying when and against which pin. A state you can set
without leaving a record is a state the next agent will silently undo.

Never carry the same fact in two fields. One of them will be wrong eventually,
and you will not know which.

**A failed verdict does not stop at the qa block.** Set the claim's
`confidence`, then walk what cited it:

- every decision whose `rationale` cites that claim id goes to
  `status: needs_review`, and its id joins `blocked_on`;
- every step listing it in `depends_on` returns to `todo`. A step listing it in
  `produces` stays `done` — the work happened, the conclusion did not survive.
  That is why those are two fields and not one: the same claim id means
  opposite things depending on which side of a step it sits.

Work continues everywhere else. It halts in exactly one case: a decision under
review with `reversible: false`. That is what the flag is for. An irreversible
decision resting on a refuted claim is the only thing worth stopping a session
over, because it is the only one you cannot cheaply undo later. A decision that
survives review returns to `active` with a rewritten rationale; one that does
not becomes `superseded`, and the correction goes in MEMORY.md.

**Verification expires.** Claims are verified against the pinned environment,
so moving the pin retires them. When any version in `environment` changes, bump
`environment.pinned_on` and append one `PIN_MOVED` verdict for every claim
whose `verified_against` no longer matches — which is what carries them to
`confidence: "stale"`. Do not just edit the field: a `stale` set without a
verdict is reverted the moment somebody re-applies the derive rule, because
the latest recorded verdict still says CONFIRMED.

Stale claims are neither deleted nor trusted — they are back in the QA queue.
Comparing `verified_against` to `pinned_on` is QA's first action on every run,
before it fetches anything.

`verified_on` does not drive that: it is the date for the record, and it
catches the other kind of drift, where the version never moved but the
documentation did. Nothing expires on it automatically — when a claim matters
and its `verified_on` is old, re-check it and say you did.

### 4. JSON is the interface

Agents communicate through task.json, not through prose summaries of each
other. This prevents compounding hallucination.

**One writer at a time.** task.json has exactly one writer: the session agent.
QA and refutation subagents *read* it and hand back per-claim verdicts —
`{id, pass, result, detail}`, the same four keys `qa.verifications` stores —
and the session agent writes them in. That structured
handback is the one thing a subagent may return under this rule, because it is
data keyed to a claim id, not a summary of the work. Run the refuters of rule 7
twenty-wide if you like; their results are merged by one writer, not by twenty.

Never run two sessions against one project's task.json. Git cannot semantically
merge a JSON array: a conflict inside `claims` is a hand-repair against two
structures, and a careless resolution drops a claim silently, leaving no trace
that it ever existed. If one does reach you, resolve it as a **union** — every
claim from both sides survives, reconciled by id. A claim that vanishes in a
merge is the exact failure this file exists to prevent.

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

- **Yes** -> it goes into your knowledge base NOW, not "later": `LAWS.md` in
  your **shared** knowledge base — one store serving every project, placed once
  (see Setup), never a per-project copy. Its absolute path belongs in MEMORY.md's
  Environment section so no session has to guess. One crisp sentence, a pointer to where the
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

### 7. Adversarial verification on anything load-bearing

Rule 3 asks: *does the source support this claim?* That catches a fabricated
source and a misquoted one. It does not catch the failure that actually gets
through — a real source, honestly cited, that does not say what the claim says.

So for any claim the work depends on, run a second pass whose job is to
**refute it**. Not to review it, not to sanity-check it: to break it. Give the
refuter the claim, its evidence, and an instruction to default to *refuted*
when it cannot prove otherwise. A claim that is true but wrong in a detail that
would mislead the person acting on it counts as refuted — say what the correct
version is.

Run the refuters independently, and where a claim can fail in more than one
way, give each a different lens (is it correct / is it complete / does it
still hold in the current version) rather than asking the same question twice.

A refuter's output goes where QA's goes: one `{id, pass, result, detail}` per
claim, handed back to the session agent, logged in `qa.verifications` with
`pass: "refutation"`. A successful refutation is `result: "REFUTED"`, which
carries the claim to `confidence: "refuted"` by the table in rule 3. The `pass`
field is why the log stays readable afterwards — "QA confirmed it, refutation
broke it" is a sequence you can read back, and it is the sequence that matters:
a claim that survived refutation is worth more than one that merely passed QA.

Why this is a rule: a README once claimed a file format was readable by four
applications. Every claim had a source and passed QA. A refutation pass found
one of the four had no support for that format at all — the string did not
appear anywhere in the application — and a second was only true via a plug-in
the docs never mentioned. Two of four compatibility claims were wrong, and
nothing in rules 1-3 was going to catch it, because the sources were real and
the citations were accurate. The claims had simply never been *tested*.

### 8. A stability metric is never reported without a paired detail metric

Any metric that rewards *the absence of change* is maximised by a result that
has nothing in it. Temporal stability is perfect on a frozen frame. Variance is
lowest on a flat one. Error rate is zero for a system that refuses to answer.

Report such a number alone and it will be optimised — by a model, by a
pipeline, or just by whoever is choosing between two options — straight towards
the degenerate case. So pair it: whenever you report a metric that punishes
change, report alongside it one that punishes emptiness, and state both or
neither.

### 9. A new metric is guilty until it convicts a known-bad input

Before a metric is allowed to inform a decision, feed it inputs you already
know are bad and confirm it ranks them badly.

The test is cheap and specific. For a video-quality metric: score a
deliberately over-smoothed clip and a completely static one. If either
outranks real footage, the metric is measuring the wrong thing and every
decision it has informed so far needs revisiting. Construct the equivalent
known-bad input for whatever you are measuring — the point is that a metric
which cannot be caught out has not been tested, it has only been used.

## Setup

### Once per machine

The knowledge base is shared by every project — that is the whole point of rule
6, and a copy per project defeats it. Place it a single time, outside any
project:

```bash
# Keep the clone. Every project is copied from it.
git clone https://github.com/AGBKdev/Agent-Work-Protocol ~/agent-work-protocol
cp -r ~/agent-work-protocol/KNOWLEDGE ~/agent-knowledge
```

Record that absolute path in every project's `task.json` `environment` block.
Versioning it is optional and cheap: `cd ~/agent-knowledge && git init -b main`.

### Once per project

```bash
cp -r ~/agent-work-protocol /path/to/project/_agent

# Two things must go, every time, or the copy is wrong in a way that is quiet:
#   .git      — otherwise `git add _agent` stores a GITLINK instead of the
#               files. It prints "adding embedded git repository", exits 0, and
#               none of this protocol lands in your repo. You find out when a
#               fresh clone of the project arrives with an empty _agent/.
#   KNOWLEDGE — the knowledge base is shared and lives at ~/agent-knowledge.
#               A per-project copy is how a cross-project knowledge base
#               quietly becomes N disjoint ones (rule 6).
rm -rf /path/to/project/_agent/.git /path/to/project/_agent/KNOWLEDGE

cd /path/to/project

# Init ONLY if not already inside a repo. Never run a bare `git init` from an
# unverified cwd — that is exactly how a repo ends up rooted at $HOME.
git rev-parse --show-toplevel 2>/dev/null || git init -b main
git rev-parse --show-toplevel        # verify: must be THIS project

# A .gitignore must exist before the first `git add -A` (rule 5). If the
# project has none, create it — naming a file that does not exist fails the
# whole command with `fatal: pathspec '.gitignore' did not match any files`.
[ -f .gitignore ] || printf '.DS_Store\n' > .gitignore

git add _agent .gitignore && git commit -m "add agent protocol"
```

Optional backup to a NAS or second disk:

```bash
rsync -av --delete /path/to/project/_agent/ \
  /path/to/backups/agent-state/PROJECT_NAME/
```
