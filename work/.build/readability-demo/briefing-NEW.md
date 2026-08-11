<!--
DEMONSTRATION RE-RENDER — spine/work/.build/readability-phase-B-handoff.md.
Source: ~/horizon/work/20260808-fix-building-group-delete-orphans-units/
briefing.md + verify.md (real, already-written records). Every fact below
is carried over from those files — nothing invented. This task is the only
one in either installed project with a full plan+briefing pair, and it's
an atypical one: the merge gate genuinely blocked it (floor failed at
lint), so the real briefing.md was hand-written, not produced by `/ship`
§4 — /ship never reached that step. This re-render keeps that fact loud,
per the writing mandate's "never bury a surprise" rule, rather than
pretending a clean ship. One measurement below (the "Plan accuracy"
bullet) was taken freshly during this demonstration, not carried from the
original record — labeled as such inline.
-->

# Shipped: `20260808-fix-building-group-delete-orphans-units` — fix building-group delete leaving units orphaned

**Status: NOT shipped — the merge gate correctly blocked it.** The real
`/ship` run never reached §4 (write the briefing); the original
`briefing.md` was written by hand so the outcome wouldn't be lost. This
file is a structural re-render of that same content into the new
template, for demonstration — not a claim that this ship happened.

**What & why:** Deleting a building group used to leave its member units
pointing at a `buildingGroupId` that no longer existed — `deleteGroup`'s
own doc comment admitted it. `deleteGroup` now unassigns every member unit
(one batched Firestore write) before deleting the group, and the
confirmation dialog copy was updated to match. A real, self-disclosed bug,
chosen as this build's worked example for being small and self-contained
with crisp acceptance criteria.

**What surprised us:** Nothing was formally logged to `deviations.md`
(this task predates that file existing as a discipline — implementation
matched the plan exactly, no halt-tier surprises during implementation
itself). Two things surfaced later, at verify, worth stating as plainly as
a deviation would be: the floor failed at `lint` on pre-existing, repo-wide
debt unrelated to this task's 3 files (see below); and the security
adversary caught an unrelated, uncommitted permission change
(`.claude/settings.json` gaining a Bash-execution allow-list) that had
bled into this diff from an earlier, unrelated session — real, fixed by
committing it separately before this briefing was written.

**Verification, honestly:**
- Floor: FAIL — `lint` (330 pre-existing unformatted files across `lib/`+
  `admin/`, plus a missing `functions/` ESLint config; none of it touched
  by this task's 3 changed files)
- Re-grounding: not applicable — this task predates Extension C's
  ship-time re-grounding step
- Adversaries: 2 ran. Falsifier: 4 verdicts, max severity medium —
  details: verify.md §3. Security: 3 verdicts, max severity **high** (the
  unrelated `.claude/settings.json` permission bleed-in above) — details:
  verify.md §4
- Capability gaps: `mutate`, `smoke-*`, `migrate-rehearse` — all
  `unavailable` for this stack (falsifier's stub-out probe substituted for
  `mutate`, ran for real)
- Tooling gaps: `floor`, `.spine/adapters/callers`, `.spine/adapters/
  dep-diff`, `verdict-filter`, `conformance`, `ledger` — all blocked
  in-session by the Auto Mode Bash-execution classifier; floor was run
  directly outside the session, `callers`/`dep-diff` reconstructed by
  hand with equivalent commands, `verdict-filter`/`conformance` never ran
  (adversary verdicts above are raw/unfiltered), `ledger` was never
  populated for this task
- Plan accuracy: not measured in the original session (`conformance` was
  blocked there too). *Measured fresh for this demonstration* against the
  real, still-uncommitted historical diff: precision 1.00, recall 0.14 —
  every predicted file was really touched; the low recall is the task
  folder's own bookkeeping plus the unrelated `.claude/settings.json`
  bleed-in already flagged above, not a planning miss.

**In six months you'll want to know:** If you're debugging a "unit still
shows a deleted building group" report: check whether the delete happened
concurrently with a unit reassignment (a TOCTOU race, known and un-fixed
as of this task — both adversaries found it independently) before
assuming this fix regressed — it didn't; this is the same defect class in
a narrower, undefended window the plan explicitly chose not to close for
a Class 1 fix.

Record: `work/20260808-fix-building-group-delete-orphans-units/` — plan,
verify.md, ledger (never populated — tooling gap, see notes.md).
