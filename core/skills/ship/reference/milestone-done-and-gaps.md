# Ship — milestone done-definition and known-gap resolution (loaded on demand)

Loaded from SKILL.md when: work/<task-id>/milestone exists.

Read `work/<id>/milestone.md`'s `## Member tasks` list. For this task's
own id, treat it as done — the merge gate (§1) already passed and §6 is
about to set `state` = `done`. For every *other* listed member task, check
its real `work/<other-task-id>/state`. If any entry is still `TBD`, or any
other member task's state isn't actually `done`, this isn't the
milestone's final ship — say nothing further, just note in passing (one
line, not a section) that member tasks remain. **If every member task is a
real id and every one is done** (by the rule above), this is the
milestone's completing ship: check the milestone's `## Done-definition`
against real state, **live, right now** — never trust
`.spine/capabilities.json`'s `implemented` flag by itself, since that flag
can go stale between whenever some earlier task set it and this exact
ship (state reads `done`, the flag reads `implemented`, and the real
capability is still broken from a cold start — a real gap a downstream
project's own M1 completion surfaced: the only reason it was caught at all
was a human asking "what's next" and reading a capability's adapter by
hand). If `## Capability targets` lists any capability, re-run conformance
for real, this exact moment, not a cached record of some earlier run:

```
${CLAUDE_SKILL_DIR}/../../scripts/adapter-conformance --all \
  --project <project root> \
  > work/<task-id>/artifacts/done-definition-conformance.txt 2>&1
```

Use *this* live result, not the cached `capabilities.json` flag, to decide
whether each listed capability is genuinely met right now —
`adapter-conformance --all` already exercises each capability's own
pass/fail/cold self-test scenarios, and `cold` specifically proves
smoke-seed/smoke-run/smoke-golden self-heal a torn-down stack rather than
assuming an earlier capability in the sequence left it running
(`core/ADAPTER-CONTRACT.md §2.2/§3.8`) — this is what actually catches the
"provably true from scratch," not "reported true once," distinction the
gap above turned on. If it could not run at all, this is the
could-not-run case in the tooling-gap discipline (`core/skills/task/
SKILL.md`'s header note): note the gap and say plainly in the briefing
that the done-definition is **unverified**, not met — never let a
could-not-run check silently read as passing. Report the result (met, not
met, or unverified) as its own section in the briefing (§4) — a milestone
that ships its final task without its done-definition actually being true
is exactly the "skeleton-skip" anti-pattern this build exists to make
impossible, so **do not let this pass silently**: if the done-definition
isn't met (or couldn't be checked), say so plainly in the briefing rather
than treating milestone completion as automatic just because every member
task individually shipped.

After reporting the done-definition result, check whether
`work/M<n+1>/milestone.md` exists (where `<n>` is this milestone's
number). If it does, say nothing — the next milestone is already planned.
If it does not, add one line to the briefing's **Milestone** section:
`"M<n> complete. No M<n+1> defined yet — run /roadmap to plan the next
slice."` This is purely informational, never a gate.

Read `work/<task-id>/plan.md`'s `## Resolves known gaps` section
(`core/templates/plan.md`, absent entirely if this plan cited none — skip
this step in that case, same as `## Grounds on decisions` in §2). For each
`gap-<n>` bullet there, remove that exact fenced entry from
`work/<id>/milestone.md`'s `## Known gaps for future member tasks` — find
by id, delete only that entry, leave `next-gap-id` and every other entry
untouched (never renumber remaining entries; a gap's id is permanent once
allocated, same reasoning `core/templates/milestone.md`'s own comment
gives for never reusing one). If a cited `gap-<n>` doesn't actually exist
in `milestone.md` (a stale citation, or a typo in the plan), don't fail
the ship over it — note it in `notes.md` ("plan cited gap-<n>, not found
in milestone.md — nothing removed") and move on; a plan-time citation
error is a plan-quality issue for a future human reader to notice, not a
merge-gate concern (§1's two checks are unchanged). This is deliberately
the mirror of §2's decision-status-flip mechanic: find by id, edit exactly
that one thing, nothing else in the file changes.
