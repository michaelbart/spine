# Task — plan, implement and ship edge cases (loaded on demand)

Loaded from SKILL.md when: the condition on the matching pointer in SKILL.md is met.

**Tooling-gap discipline (applies to every script invocation below):**
every time you invoke a core script, distinguish three outcomes — "ran and
passed," "ran and failed" (a real result, act on it normally), and **could
not run at all** (blocked, denied, or errored before the script's own logic
executed). On "could not run": append a line to `work/<task-id>/notes.md`
(create it, header `# Notes`, if it doesn't exist yet):
`TOOLING GAP: <script> could not run — <one-line consequence>.` Be concrete
about the consequence. Never let a could-not-run script silently read as
"nothing to report." This carries forward into `verify.md`'s own "Tooling
gaps" section.

**False-positive rule (own bookkeeping is not drift).** If the only drifted
items are `work/<id>/milestone.md` for this task's own milestone, and every
changed line in `git diff <sha> -- work/<id>/milestone.md` is this task's own
classify-time replacement of the milestone's first `TBD` line with its task
id (the same clause-(d) reading `core/skills/ship/SKILL.md` §0 applies at ship
time), it is not drift. Keep the research: delete the `> **STALE**` banner
`check-stale` wrote into `research.md`, append one line to `notes.md`
("check-stale flagged only my own TBD-to-task-id edit in milestone.md;
treated as a false positive, research kept"), and go on to write the plan. Any
other drifted item, or any changed line you cannot attribute to that one edit,
means regenerate as above — never guess.

**If this task does *not* already belong to a milestone** (`work/<task-id>/
milestone` unset — no `--milestone` was given), still check whether this
plan's own scope is required to make some existing milestone's `##
Done-definition` true despite not being one of that milestone's listed `##
Member tasks` — the shape a prior member task's own `briefing.md` flagging
a real gap in its follow-ups most often takes. If so, write the `##
Closes milestone gap` section per `core/templates/plan.md` naming that
milestone, then act on it right now, before presenting the plan: resolve
`work/<id>/milestone.md`, append a new numbered entry to its `## Member
tasks` with this task's own real id (no `TBD` — the task already exists)
and a one-line description drawn from `## The gist`'s first sentence, and
write `work/<task-id>/milestone` = `<id>`. Say this plainly when presenting
the plan for approval — "this also closes M<n>'s done-definition gap,
splicing it in as member task <k>" — same visibility standard as any other
milestone-affecting edit this skill makes. This is the fix for the exact
blind spot a real ad-hoc gap-closing task exposed: without it, a task that
genuinely closes a milestone's done-definition gap never gets a
`work/<task-id>/milestone` pointer, so none of `/ship`'s §3a/§3b/§3c
milestone bookkeeping ever engages for it and the milestone's own record
never shows a 5th task was actually required to reach "done." If the named
milestone doesn't resolve to a real `milestone.md`, or `work/<task-id>/
milestone` was already set to a *different* id than this section names,
that's a conflict — surface it to the human, never silently pick one or
fabricate the file.

did it teach you the plan's understanding of *the product* was wrong, or
only that this project's own tooling config (`.spine/adapters/*`,
`.spine/capabilities.json`, `.spine/protected-paths.conf`) was imperfect —
a latent adapter bug (e.g. pulling in a broken build target) with no
bearing on the plan's own reasoning, a stale capability status getting
corrected, and the like? The first is a real deviation, handled by the
three tiers below. The second is a **setup event**: fix it, append one
line to `work/<task-id>/notes.md` (create it, header `# Notes`, if it
doesn't exist yet) — `SETUP: <what was touched, what was wrong, how it was
fixed>` — and keep going. **Never a `deviations.md` record** — it doesn't
count toward the circuit breaker and doesn't appear in the briefing's
"What surprised us," because it isn't a plan-vs-reality mismatch about the
product; `core/skills/verify/SKILL.md` step 5 merges these `SETUP:` lines
into `verify.md`'s own "Setup events" section (mirroring exactly how a
`TOOLING GAP:` line already flows into that file's "Tooling gaps" section)
so it's still visible, never silent, just not conflated with a real
deviation. **A class escalation is never a setup event, even when it
traces to a plan-time check the escalation itself proves was a miss** — a
missed blast-radius call is exactly the "the plan's understanding of the
product was wrong" signal the circuit breaker exists to catch; it stays a
`halt`-tier deviation below, same as always.

- **I'll stop and ask before** (`halt`) — schema, public contracts, new
  dependencies, auth logic, or anything protected-path (the hooks enforce
  the file-level cases independently). Append a deviations.md record with
  status `open`, stop implementing, and ask the human in the form below. This is a legitimate non-recurring touchpoint — it does not
  happen on every task, only when reality diverges from the plan in a
  halt-tier way. Ask it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`). Cite no decision, gap or class
  id unless you say in a few words what it is:

  <!-- touchpoint:start -->
  > **Deciding:** <the specific thing that came up, e.g. "whether I may change the database layer">. It's yours because the plan said I'd check with you before touching <schema | public contracts | packages | login logic | protected files>.
  > **Need to know:** <what I found while building, in plain words>. <Why the plan didn't cover it>. <What each path would touch>. Nothing has been changed for this yet.
  > **Recommend:** <option> — <one-line reason>
  > 1. **<the smaller path>** — next: <what happens>; cost: <time or risk>; undo: <yes/no/how>
  > 2. **<the larger path>** — next: <what happens>; cost: <time or risk>; undo: <yes/no/how>
  > **Safe to ignore:** <records I'll update either way>
  <!-- touchpoint:end -->

**Circuit breaker:** count every deviations.md record regardless of tier.
On the third for this task, the plan is invalidated — `git stash push -u -m
"spine: circuit breaker, work/<task-id>"` to preserve what you'd built
without losing it, run `${CLAUDE_SKILL_DIR}/../../scripts/set-state <task-id> research`, and tell the human plainly: three wrong guesses means the
research was wrong once, not that each guess should be patched forward.
Fresh research is required before re-planning.

If a resolution (halt or otherwise) cites a `docs/charter.md` line, it must
end amend-or-reaffirm: the human either edits that charter line or
reaffirms it as-is, dated, and the deviations.md resolution records which.
