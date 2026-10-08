# Task — auto autonomy steps (loaded on demand)

Loaded from SKILL.md when: `work/<task-id>/autonomy` is `auto`.

**If `work/<task-id>/autonomy` is `auto`, there is no plan-approval stop.**
Write the plan exactly as above — it is still written, and `/ship` attaches it
to the PR for review, trading pre-implementation plan review for PR-time review
(the disclosed `auto` tradeoff — see `docs/tradeoffs.md`). Record `approval.json` with `"autonomy": "auto"` set,
and proceed straight to §4. This can only happen at Class 1
(the ceiling); if §3's protected-path check just auto-escalated this task to
Class 2, `work/<task-id>/autonomy` was set to `guided` above, so this branch no
longer applies and you fall through to the stop below. For `guided`:

**auto** — no scheduled human stop. **Follow `core/skills/verify/SKILL.md` inline**
(the falsifier's stub-out probe is *mandatory*
in this mode — it is the partial backstop for the plan review `auto` skipped).
Read `verify.md`'s
`Result:`. On `PASS`, **follow `core/skills/ship/SKILL.md` inline**, which for an
`auto` task opens a **draft PR** (never merges — `core/skills/ship/SKILL.md` §5a)
and stops at "PR ready for review." On `FAIL`, this is an
*exception* stop: go back to implementation, fix, re-run verify inline; if the fix
hits a `halt`-tier decision or trips the circuit breaker (§4), stop and pull the
human in exactly as §4 says. The human's single touchpoint is reviewing the
finished PR.
