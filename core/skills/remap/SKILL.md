---
name: remap
description: Regenerate docs/map.md from the current repository state, stamped with the current commit SHA. Runs isolated from the calling conversation.
disable-model-invocation: true
context: fork
---

Survey this repository and rewrite `docs/map.md` from
`${CLAUDE_SKILL_DIR}/../../templates/map.md` (hand this path to the shell
verbatim, `../../` included — do **not** lexically collapse it to
`.claude/`, which is a symlink into the spine core checkout) — real module
boundaries, real data flow for the flows that matter most (entry to
persistence and back, not every path), real entry points, and a
`## Known weirdness` section stating plainly what you find (half-finished
migrations, undocumented invariants, dead code, naming drifted from
reality) — surfaced, not fixed here.

This is a survey, not an audit: bounded depth, the same discipline
`/adopt`'s initial survey uses. If you notice you're going deep enough that
this looks like task-level research into one specific area, stop there —
that depth belongs in a `/task`'s research phase, not here.

Stamp the header with `git rev-parse HEAD` and the current time, in exactly
the format the template shows (`sha:` / `generated:` — this isn't machine-
parsed by a script the way `research.md`'s header is, but the research
skill reads it to judge the map's age, so keep the format consistent).

You have no access to the conversation that invoked you — everything you
need is the repository on disk. Write the file directly; your final reply
should be a short summary of what changed since the map's previous stamp
(new/removed module boundaries, newly-noticed weirdness), not the full
map contents again.
