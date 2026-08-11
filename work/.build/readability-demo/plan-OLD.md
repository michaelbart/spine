# Plan: Fix building group delete orphans units

Task ID: `20260808-fix-building-group-delete-orphans-units`
Class: 1
Research: `work/20260808-fix-building-group-delete-orphans-units/research.md`

## Problem

`BuildingGroupsCubit.deleteGroup` (`lib/features/properties/building_groups/cubit/building_groups_cubit.dart:87-92`) deletes the group document but never touches its member units, so any unit that had `buildingGroupId` set to the deleted group's id is left pointing at a nonexistent group. The method's own doc comment acknowledges this. The UI's confirmation dialog also currently tells the user reassignment is a manual, separate step.

## Approach

Unassign the group's units *before* deleting the group document (not atomically/transactionally — see Rejected alternatives). This ordering means a failure at either step never produces a dangling reference: if unassignment fails, the group and its members are untouched (status quo); if unassignment succeeds but the subsequent delete fails, units are correctly unassigned and the group still exists (just membership went to zero) — no orphaned `buildingGroupId` in either failure case.

Add a new batched repository method `removeUnits` (mirroring the existing `assignUnits` batch pattern) rather than looping single-unit `removeUnit` calls, so the unassignment is one Firestore batch write consistent with how multi-unit assignment already works in this same file.

## Steps

1. **`lib/repositories/building_groups_repository.dart`** — add `removeUnits(List<String> unitIds)`, placed next to `assignUnits`/`removeUnit`. Batched `FieldValue.delete()` update per unit (same shape as `assignUnits`, using `removeUnit`'s field-clearing update), wrapped in `RetryHelper.retryFirestoreOperation` and the same `FirebaseException` → `ServerFailure` catch used by every other method in this file. No-op (return `Right(null)` immediately) when `unitIds` is empty, to avoid committing an empty batch.

2. **`lib/features/properties/building_groups/cubit/building_groups_cubit.dart:87-92`** — rewrite `deleteGroup`:
   - Fetch member units via `_unitsRepository.getUnitsByBuildingGroup(group.id)`.
   - On failure, return `failure.message` (abort — do not delete the group).
   - On success with a non-empty unit list, call `_buildingGroupsRepository.removeUnits(unitIds)`. On failure, return `failure.message` (abort — do not delete the group; nothing has been orphaned).
   - Then call `_buildingGroupsRepository.delete(group.id)` as today, returning its failure message or `null`.
   - Update the doc comment to state units are unassigned as part of deletion (remove the "does not unassign" disclaimer).

3. **`lib/features/properties/building_groups/view/building_groups_page.dart:179-182`** — update the confirmation dialog copy to no longer claim manual reassignment is needed (e.g. "Delete "${group.name}"? Units currently assigned to it will be unassigned."), since that's now false.

## Predicted touch

- `lib/repositories/building_groups_repository.dart`
- `lib/features/properties/building_groups/cubit/building_groups_cubit.dart`
- `lib/features/properties/building_groups/view/building_groups_page.dart`

## Rejected alternatives

- **Atomic batch covering both the group-doc delete and unit updates**: Firestore batched writes support mixing a `delete()` and `update()`s in one batch, which would make this fully atomic. Rejected for this fix because no other method in `BuildingGroupsRepository` composes a delete with cross-collection updates in one batch, and the two-step ordering above already eliminates the orphan outcome (the one bug being fixed) without introducing a new cross-cutting batch pattern for a Class 1 fix. Worth revisiting if a future task needs stronger atomicity guarantees here.
- **Loop `removeUnit` per unit from the cubit** instead of a new `removeUnits` batch method: rejected because it means N sequential network round-trips instead of one batch, and diverges from the `assignUnits` batching convention already established in the same file for the symmetric "many units" case.
- **New test file for `BuildingGroupsCubit`**: no test exists today for this cubit or `BuildingGroupsRepository` (only a template-default `counter_cubit_test.dart` exists in the whole repo), so there's no established pattern to extend. Adding cubit-mocking test infrastructure from scratch is out of scope for a small, self-contained bug fix; verification instead relies on `flutter analyze` plus manual exercise of the delete flow (see Acceptance checks).

## Acceptance checks

- `flutter analyze` reports no new errors/warnings.
- Reading the new `deleteGroup` implementation confirms: units are fetched and unassigned before the group document is deleted, and any repository failure at either step returns a non-null error without leaving a partially-orphaned state.
- Manual check (via `debug main chrome` or `flutter run`): create a building group, assign a unit to it, delete the group, then confirm in the unit's detail/edit view that it no longer shows the deleted group (i.e. `buildingGroupId` was cleared) rather than a stale/broken reference.
- Confirmation dialog text no longer claims manual reassignment is required.

## Latitude for implementation

- **Decide-alone**: exact wording of the updated confirmation dialog copy; exact log message strings in the new `removeUnits` method (follow existing conventions in the file).
- **Record-and-proceed**: if `getUnitsByBuildingGroup` or `removeUnits` needs a different empty-list short-circuit shape than assumed above once the code is in front of me.
- **Halt**: anything requiring a Firestore rules/index change, a new public API contract beyond what's described above, or touching a protected path.
