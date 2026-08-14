# Agent Work Protocol

A three-file system for doing serious, multi-session work with AI agents
without the two failure modes that kill long projects: **compounding
hallucination** (agents summarizing agents until nobody can say where a
"fact" came from) and **paid-for knowledge evaporating** (findings buried
in chat transcripts that the next session re-derives at full price).

It came out of months of AI-assisted production work — a 3D-to-photoreal
pipeline built with Claude across dozens of sessions, hundreds of verified
claims, and enough wrong turns to learn exactly where agent workflows leak.

## The idea

Split agent state into three files with different authority:

| File | Role | Authority |
|---|---|---|
| `task.json` | Atomic, sourced, checkable claims + decisions + QA state | **Source of truth** |
| `MEMORY.md` | Human-readable state for session pickup and handoff | Commentary |
| `PROTOCOL.md` | The rules | The rules |

Everything consequential an agent asserts becomes a **claim**: one
checkable statement, with a source, tagged `confidence: "unverified"` until a
**separate QA pass** — which reads *only* task.json, never the chat — has
fetched the source and confirmed it. Agents hand off through the JSON, not
through prose summaries of each other.

Anything the work depends on then gets a second, **adversarial** pass whose job
is to refute it. QA catches a claim whose source is fake or misquoted; only
refutation catches the commoner one, where the source is real and accurate and
still does not say what the claim says.

At the end of every session, any finding that would hold on a *different*
project gets promoted to a cross-project knowledge base
(`KNOWLEDGE/LAWS.md`) immediately — with an id, evidence pointer, and a
confidence tag (`measured` / `single observation` / `reasoned, not
measured`). Contradicted laws are marked superseded, never deleted.

## Quickstart

0. **Once, on first adoption:** place `KNOWLEDGE/` outside any project, at a
   path all your projects can reach (see PROTOCOL.md Setup). It is one shared
   store, not a per-project copy — that is what makes rule 6 pay.
1. Copy the rest of this folder into your project (e.g. as `_agent/`).
2. Fill in `task.json`'s `environment` block, including `pinned_on` — that
   is the authoritative pin. Claims are verified against those versions and
   retired when they move (PROTOCOL.md rule 3), so a placeholder left in
   `pinned_on` means nothing ever goes stale.
3. Point your agent at `PROTOCOL.md` at the start of every session
   (in Claude Code, reference it from `CLAUDE.md`).
4. Work. Claims accumulate in `task.json`.
5. Run QA as a separate agent/subagent whose only input is `task.json`.
6. End of session: update `MEMORY.md`, promote transferable findings to
   `KNOWLEDGE/LAWS.md`, commit — after verifying the repo root
   (`git rev-parse --show-toplevel`; see PROTOCOL.md rule 5 for why).

## What's in the box

- `PROTOCOL.md` — the nine rules, with the war stories that made them rules
- `MEMORY.md` — annotated template
- `task.json` — annotated template with the claim/QA schema and one worked example
- `KNOWLEDGE/LAWS.md` — knowledge-base seed with the law format; placed once,
  shared by every project, never copied per project

## Does it work?

The project this system was extracted from ran a six-agent QA sweep over
its accumulated claims: every source URL re-fetched, ~105 claims confirmed,
~21 partially supported, 5 unsupported (and therefore caught), zero
fabricated URLs, zero dead links. The unsupported ones are the point — the
system exists so those five get caught in QA instead of shipping.

## When it's worth it, and when it isn't

Worth the overhead: multi-session projects; licensing, compatibility and
version assertions; anything a client sees; anything you will be asked to
justify in six months.

Not worth it: one-off scripts, exploratory spikes, throwaway work. Running the
full claim/QA loop on a thirty-minute experiment costs more than re-deriving
the answer.

The cost scales with claim count, not project size, and two things bound it:
rule 1 limits what becomes a claim in the first place (API signatures, config
syntax, version behaviour, licensing terms — not every sentence an agent
writes), and rule 7's adversarial pass applies only to the load-bearing ones.

The sweep quoted above was six agents over 131 claims. We have not measured
agent-hours or tokens, so we are not going to quote a figure — this protocol's
own `measured` / `single observation` / `reasoned, not measured` tags exist
precisely to stop unmeasured numbers being presented as measured.

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — © 2026 Axel Gimenez.

Use it, adapt it, ship it with your own projects, commercially or not. The one
condition is attribution: credit the source and say if you changed it. Keeping
this README's title line in your copy of `_agent/` is enough.

A Creative Commons licence rather than a software one because what is being
licensed here is writing — rules, templates and the reasoning behind them. It
was Apache-2.0 until v1.0, which is a source-code licence whose terms are about
object code, patent grants and NOTICE files. None of that describes prose.
