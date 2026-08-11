<!--
DEMONSTRATION — spine/work/.build/pr-patch-phase-B-handoff.md.
Source: ~/horizon/work/20260808-fix-building-group-delete-orphans-units/
verify.md (real, already-written record) + the readability patch's own
plan-NEW.md re-render (spine/work/.build/readability-demo/, itself sourced
from the same task's real plan.md — reused here rather than re-condensing
the same prose a second time). Every fact below traces to one of those.

**Status: this task's real `/ship` run never reached §4a — the merge gate
correctly blocked it at the floor (lint FAIL, pre-existing debt).** This
is what `/ship`'s new PR-description step *would have produced* had the
floor passed, assembled from the frozen real record, not a claim that a PR
was actually opened. Same disclosure the readability patch already made
re-rendering this task's plan/briefing.
-->

## 20260808-fix-building-group-delete-orphans-units — fix building-group delete leaving units orphaned

**What & why:** Deleting a building group used to leave its member units
pointing at a `buildingGroupId` that no longer existed — `deleteGroup`'s
own doc comment admitted it. `deleteGroup` now unassigns every member unit
(one batched Firestore write) before deleting the group, and the
confirmation dialog copy was updated to match.

**Review this at the plan level:** A two-step fix — fetch member units,
remove their group reference in one batch, then delete the group — chosen
over a fully atomic batch because no other method in this repository
composes a delete with cross-collection updates in one batch, and the
ordering already eliminates the orphan outcome without a new cross-cutting
pattern for a Class 1 fix. Two real risks named in the plan: the two-step
sequence isn't atomic (a mid-sequence failure leaves units unassigned and
the group intact — never an orphan, an accepted trade-off); and neither
`BuildingGroupsCubit` nor `BuildingGroupsRepository` has any existing test
coverage, so verification leans on `flutter analyze` plus manual exercise
rather than an automated regression check.

**How it was verified:**
- Floor: **FAIL** — `lint` (`dart format` reformats 330 pre-existing files
  across `lib/`+`admin/`, and `functions/` has no committed ESLint config
  at all; none of it touched by this task's 3 changed files). This is the
  sole reason for FAIL.
- Adversaries: Falsifier — 4 verdicts, max severity **medium**. Security —
  3 verdicts, max severity **high** (an unrelated, uncommitted
  `.claude/settings.json` permission-escalation change bled into this
  diff from an earlier session). Details: verify.md.
- Plan accuracy: not computable for this record — `plan.md`'s `##
  Predicted touch` entries are Markdown-backtick-wrapped (this task
  predates the readability patch's fix to that template), so a raw
  `conformance` set difference would falsely count every genuinely-
  predicted file as drift too. Not run in the original session either
  (sandbox-blocked, per verify.md's own disclosure).
- Gaps: `mutate`/`smoke-*`/`migrate-rehearse` — all `unavailable` for this
  stack (falsifier's stub-out probe substituted for `mutate`, ran for
  real). Tooling gaps: `floor` (run directly outside the session instead),
  `.spine/adapters/callers`/`dep-diff` (reconstructed by hand),
  `verdict-filter`/`conformance` (blocked, never ran — adversary verdicts
  below are raw/unfiltered), `ledger` (never populated for this task).

**Where to look (in priority order):**
- `lib/features/properties/building_groups/cubit/building_groups_cubit.dart:91`
  — a TOCTOU race found independently by both adversaries: `deleteGroup`
  fetches member units once and never re-checks before deleting the
  group; a unit assigned concurrently in that window is never included in
  the unassign batch and ends up orphaned anyway — the same defect class
  this task exists to fix, narrowed to a race window the plan explicitly
  chose not to close.
- `lib/repositories/building_groups_repository.dart:250` — the new
  `removeUnits` only catches `on FirebaseException`, no generic
  `on Exception` fallback (unlike `getByProperty`/`getByProperties` in the
  same file); a non-`FirebaseException` throw during the batch write
  propagates unhandled. Pre-existing gap in a sibling method, inherited
  verbatim by the new code.
- `firestore.rules:87` — pre-existing lack of per-organization scoping on
  `buildingGroups`/`units` (`read, write: if request.auth != null`); this
  diff's new unassign step widens a single-document cross-tenant write
  into a batched cross-collection one over the same open rule, with no
  new tenant check added to compensate.
- plus 3 finding(s) without file anchors — see verify.md. (One of these is
  the run's own max-severity finding — the `.claude/settings.json`
  permission-escalation change — visible in full above under
  "Adversaries," not lost, just without a file:line to anchor here: its
  evidence is a `git diff` command, not a file location.)

**What surprised us:** No `deviations.md` exists for this task (it
predates that discipline as a hard requirement — implementation matched
the plan with no halt-tier surprises along the way). Two things surfaced
at verify instead, stated as plainly as a deviation would be: the floor
failed on pre-existing, unrelated lint debt (above), and the security
adversary caught an unrelated, uncommitted permission change that had
bled into this diff from an earlier session.

Record: `work/20260808-fix-building-group-delete-orphans-units/` — plan,
verify.md, notes.md (ledger never populated — tooling gap).
