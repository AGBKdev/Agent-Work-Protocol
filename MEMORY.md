# MEMORY - <project name>

> Read this first. Then read `task.json`, which is the source of truth for
> claims and verification state. If this file and task.json disagree,
> task.json wins.

## Current state (read this before anything else)

<Two or three paragraphs a fresh session needs before touching anything:
what the project is, what approach is CURRENT, and — just as important —
which previously-built approaches have been abandoned and why, so nobody
resurrects them. Date the pivots.>

## Environment

<A human-readable echo of `task.json`'s `environment` block, plus the path
to your shared knowledge base. task.json holds the authoritative pin — this
is commentary, and if the two ever disagree, task.json wins. Example
entries:>

- **Workstation:** <OS, GPU, driver>
- **<Main tool>** <version> at <path>, <how it's launched>
- **<Runtime>** <python/node/etc. version>, <key packages + versions>
- **Storage:** <where inputs live> == <where the other machine sees them>

## Done

<Reverse-chronological session log. Each session: a dated heading, then
findings as bullets. State WHAT was measured, HOW, and what it rules out.
A finding that would hold on a different project does not belong here —
promote it to the knowledge base (PROTOCOL.md rule 6) and reference its id.>

## Next

<Ordered list of what the next session should do first.>

## Corrections

<When a recorded conclusion turns out to be wrong, do not silently edit
history. Add a dated correction notice here (or at the top if it poisons
many entries below), stating what was wrong, what the fixed understanding
is, and which downstream conclusions need re-measuring.>
