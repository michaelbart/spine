# Ship — re-grounding drift cross-check (loaded on demand)

Loaded from SKILL.md when: check-stale reported STALE for one or more files.

**Before treating STALE as real drift, cross-check each drifted file
against this task's own record** (disclosed fix — the naive "any STALE
verdict is drift" rule produces a guaranteed false positive on every task
that touches a file it also read as grounding, which is most tasks). A
drifted file is
**expected, not drift**, if any of the following is true: (a) it appears
in `work/<task-id>/plan.md`'s own `## Predicted touch` list — this task's
own approved implementation changed it; (b) it's
explained by a `work/<task-id>/deviations.md` record with `Status:
resolved` (e.g. a manifest file changed because an approved
new-dependency deviation added one); or (c) it's named in
`work/<task-id>/verify.md`'s own "Adversary verdicts" section against a
kept finding marked `FIXED` there — an adversary-found fix applied and
recorded during `/verify` is exactly as "this task's own approved work"
as (a)/(b), and forcing every such fix through
`deviations.md` too would make the circuit breaker fire on legitimate,
already-adversarially-verified work with nothing left to stash (disclosed
fix); or (d) it is `work/<id>/milestone.md` for the
milestone this task itself belongs to (`work/<task-id>/milestone` names
`<id>`), and every changed line `git diff <sha> -- work/<id>/milestone.md`
shows traces to this task's own mandated bookkeeping in that shared file —
the classify-time replacement of the milestone's first `TBD` member-task
line with this task's own id (`core/skills/task/SKILL.md`'s `--milestone`
header note, made before research even ran) and/or a `## Known gaps for
future member tasks` entry whose `source` field cites this task's own
`work/<task-id>/verify.md` (the edit §3a below makes at ship time,
possibly made early during `/verify` under the same triage convention). A
milestone-tagged task's `research.md` citing `work/<id>/milestone.md` as
grounding — the expected case, since `core/skills/task/SKILL.md` loads it
as planning context for research to read — makes this guaranteed on the
classify-time edit alone, on every such task, not an edge case (disclosed
fix). A changed
line that isn't one of those two things — another member task's own `TBD`
slot resolved, a different task's Known-gaps entry, an edit to
`## Capability targets` — is a real change and stays fully driftable; (d)
accounts for only this task's own hand in the milestone file, never the
file wholesale. Only a file covered by
**none of the four** is genuine unexplained drift. If every drifted file
is expected by this rule: proceed normally — not a halt, not a deviation,
does not touch the circuit breaker.

If any drifted file is **not** covered by any of these checks: this is a real
deviation, not a soft warning — append a
`work/<task-id>/deviations.md` record, tier `halt` (grounding drifted
since this was last verified, the same halt tier already assigned to
schema/auth surprises), and **it counts
toward the circuit breaker** (`core/skills/task/SKILL.md`'s existing
three-deviation rule) — decided and defended here, not left as an open
question: the circuit breaker's actual trigger condition is "the research
this plan stands on is no longer trustworthy," which is exactly as true
when the code moved underneath it as when the original research was
simply wrong.
Proceeding means going back to `core/skills/task/SKILL.md` step 2
(research), not continuing here.
