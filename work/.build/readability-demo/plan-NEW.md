<!--
DEMONSTRATION RE-RENDER — spine/work/.build/readability-phase-B-handoff.md.
Source: ~/horizon/work/20260808-fix-building-group-delete-orphans-units/
plan.md (real, already-written task record). Every fact below is carried
over from that file or its sibling research.md/verify.md — nothing here is
invented. Two honest gaps, disclosed rather than papered over:
  - `owner:` is unrecorded — this task predates Extension C's
    work/<task-id>/owner file, so there is no real git identity to put
    there. Written as "unrecorded" rather than guessed.
  - research.md's own grounding is marked STALE as of this re-render
    (files changed since sha 94c93e27...) — carried into the grounding
    line verbatim, since that's the honest current state, not the state
    at original plan time.
-->

# Plan: `20260808-fix-building-group-delete-orphans-units` — fix building-group delete leaving units orphaned

<!-- MACHINE: header -->
task: 20260808-fix-building-group-delete-orphans-units   class: 1   owner: unrecorded (task predates Extension C)   milestone: none
grounding: research `94c93e2705851cff3f5e07de53592b6269eac5ca` (`work/20260808-fix-building-group-delete-orphans-units/research.md`, now STALE as of this re-render)
<!-- /MACHINE -->

## The gist

Deleting a building group currently leaves its member units pointing at a
`buildingGroupId` that no longer exists — `BuildingGroupsCubit.deleteGroup`
deletes the group document but never touches its units, and its own doc
comment admits it. This task makes `deleteGroup` unassign every member unit
(one batched Firestore write, added as a new `removeUnits` repository
method mirroring the existing `assignUnits` batch pattern) *before*
deleting the group document, so neither a partial failure nor the happy
path can ever leave a unit with a dangling reference. The confirmation
dialog's copy is updated to match, since it currently tells users
reassignment is a manual step that will no longer be true. Rejected: an
atomic Firestore batch spanning both the unit updates and the group
delete — no other method in this repository composes a delete with
cross-collection updates in one batch, and the two-step ordering already
eliminates the orphan outcome without introducing a new cross-cutting
batch pattern for a Class 1 fix.

## What could go wrong

- The two-step (unassign-then-delete) ordering is not atomic. A failure
  between the two steps is handled — it leaves units unassigned and the
  group still existing, never an orphan — but this is a deliberate
  trade-off against a fully atomic batch, made explicitly to stay in scope
  for a Class 1 fix (see the gist's rejected alternative).
- Neither `BuildingGroupsCubit` nor `BuildingGroupsRepository` has any
  existing test coverage — no established pattern to extend without
  building cubit-mocking infrastructure from scratch, out of scope here.
  Verification leans on `flutter analyze` plus a manual exercise of the
  delete flow, not an automated regression check.

## What I'll decide alone vs. stop and ask

**I'll just do** (`decide-alone`):
- Exact wording of the updated confirmation dialog copy; exact log message
  strings in the new `removeUnits` method (follow existing conventions in
  the file).

**I'll do and note** (`record-and-proceed`):
- If `getUnitsByBuildingGroup` or `removeUnits` needs a different
  empty-list short-circuit shape than assumed once the code is in front of
  me.

**I'll stop and ask before** (`halt`):
- Anything requiring a Firestore rules/index change, a new public API
  contract beyond what's described above, or touching a protected path.

## Steps

1. **`lib/repositories/building_groups_repository.dart`** — add
   `removeUnits(List<String> unitIds)` next to `assignUnits`/`removeUnit`:
   batched `FieldValue.delete()` update per unit, same shape as
   `assignUnits`, wrapped in `RetryHelper.retryFirestoreOperation` and this
   file's standard `FirebaseException` → `ServerFailure` catch; no-op
   (`Right(null)`) on an empty `unitIds` rather than committing an empty
   batch. — **acceptance:** `flutter analyze` reports no new errors/
   warnings, and the method matches `assignUnits`' batching convention.
2. **`lib/features/properties/building_groups/cubit/building_groups_cubit.dart:87-92`**
   — rewrite `deleteGroup`: fetch member units via
   `_unitsRepository.getUnitsByBuildingGroup`; on failure, abort and return
   the failure message without deleting the group; on success with units,
   call `removeUnits`, aborting the same way on failure; only then call
   `_buildingGroupsRepository.delete`; update the doc comment to drop the
   "does not unassign" disclaimer. — **acceptance:** reading the
   implementation confirms units are fetched and unassigned before the
   group document is deleted and any repository failure returns a
   non-null error without a partially-orphaned state; manually creating a
   group, assigning a unit, deleting the group, and checking the unit's
   detail view confirms `buildingGroupId` was cleared rather than left
   dangling.
3. **`lib/features/properties/building_groups/view/building_groups_page.dart:179-182`**
   — update the confirmation dialog copy to no longer claim manual
   reassignment is needed. — **acceptance:** dialog text no longer claims
   manual reassignment is required; `flutter analyze` reports no new
   errors/warnings.

<!-- MACHINE: predicted-touch -->
## Predicted touch

- lib/repositories/building_groups_repository.dart — new batched removeUnits method
- lib/features/properties/building_groups/cubit/building_groups_cubit.dart — rewritten deleteGroup
- lib/features/properties/building_groups/view/building_groups_page.dart — confirmation dialog copy
<!-- /MACHINE -->

## How we'll know it worked

`flutter analyze` plus a direct read of the new `deleteGroup` ordering
demonstrate the failure-handling invariant; a manual create-assign-delete
exercise demonstrates the fix's actual user-visible effect. Verification
cannot show automated regression protection this time — there is no
existing test coverage for this cubit or repository to extend, and adding
cubit-mocking test infrastructure from scratch was ruled out of scope for
this Class 1 fix (see the gist). `mutate`, `smoke-*`, and
`migrate-rehearse` are all `unavailable` for this stack already
(`.spine/capabilities.json`) and were not expected to run regardless.
