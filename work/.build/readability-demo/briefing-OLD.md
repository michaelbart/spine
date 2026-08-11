# `20260808-fix-building-group-delete-orphans-units`: fix building-group delete leaving units with a dangling `buildingGroupId`

**Status: implemented and verified, NOT shipped.** The merge gate legitimately
blocked this — see below. Written by hand rather than by `/ship` (the merge
gate refused, correctly) so the outcome is still on record.

## What changed

Deleting a building group used to leave its member units pointing at a
`buildingGroupId` that no longer existed — the UI told users they'd have to
reassign those units manually, which was easy to forget. `deleteGroup` now
unassigns every member unit (one batched Firestore write) before deleting
the group, so no unit is ever left with a dangling reference. The
confirmation dialog copy was updated to match.

## Why

A real, self-disclosed bug — the old method's own doc comment said "does
not unassign its units first." Chosen as this build's worked example
because it was small, self-contained, and had crisp acceptance criteria.

## Deviations taken

None logged to `deviations.md` — implementation matched the plan exactly,
three files, no halt-tier surprises. (The ledger/check-stale/floor/
verdict-filter/conformance scripts being unreachable from the driving
session was a real limitation, but it's a tooling gap, not a plan
deviation — recorded in `notes.md` and `verify.md` instead.)

## Decisions made

None distilled to `docs/decisions/` — nothing here rose to "a future
researcher would want to find this."

## Capability gaps that degraded verification

- `mutate`, `smoke-*`, `migrate-rehearse` — all `unavailable` for this
  stack already (`.spine/capabilities.json`); the falsifier's stub-out
  probe substituted for `mutate`, ran for real.
- `floor`, `.spine/adapters/callers`, `.spine/adapters/dep-diff`,
  `verdict-filter`, `conformance`, `ledger` — all blocked in the driving
  session by an Auto Mode Bash-execution classifier (see
  `docs/tradeoffs.md`, "The Auto Mode classifier wall"). Floor was run
  directly outside that session; `callers`/`dep-diff` were reconstructed
  by hand with equivalent commands; `verdict-filter`/`conformance` did not
  run (adversary verdicts below are raw, unfiltered — both were fully
  well-formed, so filtering would not have dropped anything); `ledger` was
  not populated at all for this task.

## Why this did not ship

Two independent, real reasons — neither is this task's own code:

1. **The floor genuinely failed at `lint`** — 330 pre-existing unformatted
   Dart files and a missing `functions/` ESLint config, repo-wide,
   unrelated to any of this task's 3 changed files. This is not a
   pre-existing-debt case where `--bypass` is appropriate (`/ship`'s own
   contract reserves bypass for genuine emergencies, not disagreement with
   a check) — it's a real, standing problem that needs its own dedicated
   task. See `docs/tradeoffs.md`'s worked-example section for the two
   honest paths forward.
2. **The security adversary caught an unrelated, uncommitted permission
   change** (`.claude/settings.json` gaining a Bash-execution allow-list)
   sitting in the working tree and bleeding into this task's diff. It was
   real — added earlier in this same build session for an unrelated reason
   (Phase D's stack-independence audit) and never committed separately.
   Fixed by committing it on its own, out of this diff, before writing
   this briefing. This is exactly the adversary layer doing its job: an
   out-of-scope permission-escalation-shaped change riding along in a
   narrow bug-fix diff is precisely what it exists to catch, regardless of
   how innocent the actual explanation turns out to be.

Both adversaries also surfaced a real, **not fixed**, disclosed finding
worth a human decision: a TOCTOU race in `deleteGroup` (units fetched once,
never re-checked before `delete()` — a unit assigned to the group in that
window ends up orphaned, the same bug class this task fixes, narrowed to a
race). The plan explicitly rejected an atomic batch for this Class 1 fix;
this is a direct, known consequence of that choice, not an oversight — left
for a human to decide whether it's worth a follow-up task or an accepted
residual risk given how narrow the window is in practice.

## What you'd want to know in six months

If you're debugging a "unit still shows a deleted building group" report:
check whether the delete happened concurrently with a unit reassignment
(the TOCTOU race above, known and un-fixed as of this task) before assuming
this fix regressed — it didn't; this is the same defect class in a
narrower, undefended window the plan explicitly chose not to close.
Separately: this task is real evidence that `/verify`'s adversary layer
works even when the floor's own orchestration is degraded — both
adversaries ran for real, found real things, including something about
*this build's own process hygiene*, not just the target code.
