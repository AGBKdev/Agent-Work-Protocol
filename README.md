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
checkable statement, with a source, tagged `verified: false` until a
**separate QA pass** — which reads *only* task.json, never the chat — has
fetched the source and confirmed it. Agents hand off through the JSON, not
through prose summaries of each other.

At the end of every session, any finding that would hold on a *different*
project gets promoted to a cross-project knowledge base
(`KNOWLEDGE/LAWS.md`) immediately — with an id, evidence pointer, and a
confidence tag (`measured` / `single observation` / `reasoned, not
measured`). Contradicted laws are marked superseded, never deleted.

## Quickstart

1. Copy this folder into your project (e.g. as `_agent/`).
2. Fill in `MEMORY.md`'s Environment section — pin exact versions;
   claims are verified against them.
3. Point your agent at `PROTOCOL.md` at the start of every session
   (in Claude Code, reference it from `CLAUDE.md`).
4. Work. Claims accumulate in `task.json`.
5. Run QA as a separate agent/subagent whose only input is `task.json`.
6. End of session: update `MEMORY.md`, promote transferable findings to
   `KNOWLEDGE/LAWS.md`, commit — after verifying the repo root
   (`git rev-parse --show-toplevel`; see PROTOCOL.md rule 5 for why).

## What's in the box

- `PROTOCOL.md` — the six rules, with the war stories that made them rules
- `MEMORY.md` — annotated template
- `task.json` — annotated template with the claim/QA schema and one worked example
- `KNOWLEDGE/LAWS.md` — knowledge-base template with the law format

## Does it work?

The project this system was extracted from ran a six-agent QA sweep over
its accumulated claims: every source URL re-fetched, ~105 claims confirmed,
~21 partially supported, 5 unsupported (and therefore caught), zero
fabricated URLs, zero dead links. The unsupported ones are the point — the
system exists so those five get caught in QA instead of shipping.

## License

Apache-2.0. Use it, adapt it, ship it with your own projects.
