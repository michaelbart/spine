# Phase E Handoff — Three Pre-Use Fixes

Date: 2026-08-08. Builder: Claude Code (Sonnet 5), interactive session
(fresh session from Phase D per the standing session-boundary discipline).

Read order for anyone resuming or auditing: `phase-D-handoff.md`, this
file, then `docs/tradeoffs.md` (the fuller written record — this handoff
points at specific sections rather than duplicating them), then the state
of `/Users/michaelbart/spine` and `/Users/michaelbart/horizon` on disk.

This phase was not asked to design anything new. All three fixes come from
findings Phase D's own tradeoffs document and self-red-team already named
and, in two of three cases, explicitly declined to resolve unilaterally.
Nothing here was invented; it was decided and closed.

## 1. What was fixed

### Fix 1 — Closed the Bash bypass in `phase-gate`/`path-escalate`

**Before:** both hooks matched `Edit|Write` only. A Bash shell redirect
(`printf ... > file`, `sed -i`) bypassed both entirely — the single most
exploitable gap in the enforcement layer per Phase B and Phase D's own
self-red-team.

**What changed:**
- New shared library `core/hooks/_bash-write-targets` (sourced by both
  hooks, not a hook itself, not wired in any settings.json). Recognizes
  output redirection, `sed`/`perl -i`, `tee`, `cp`/`mv`/`install`,
  `truncate`, `dd of=` — extracts candidate write targets from a Bash
  command string via regex/token heuristics (explicitly not a real shell
  parser).
- Both hooks widened to `Edit|Write|Bash`, each resolving Bash targets the
  same way they already resolved Edit/Write's `file_path` — same task-
  folder check (`phase-gate`), same protected-path glob match
  (`path-escalate`).
- **Fails closed**: a write-shaped command whose target can't be
  confidently resolved (shell variable, command substitution, glob, or a
  relative path after an in-command `cd`) is denied outright, not passed
  through. `/dev/null`/`/dev/stdout`/etc. are explicitly carved out as
  non-mutating idioms, not silently ignored as "can't resolve."
- `core/skills/bootstrap/SKILL.md §5` updated (all three hooks now wire as
  `Edit|Write|Bash`; `adopt/SKILL.md` inherits this via its own §5
  reference, no separate edit needed).
- horizon's real, installed `.claude/settings.json` updated to match —
  this took effect immediately since `.claude/hooks` is a whole-directory
  symlink into `spine/core/hooks`.

**Watched firing, four fresh `claude -p` subprocesses against horizon's
real tree** (per the standing "a hook you have not watched fire does not
exist" discipline):
1. `printf 'x' > lib/bash_bypass_firetest.dart` during a scratch task's
   research phase — real block by `phase-gate`; file never created.
2. `sed -i '' 's/nonexistentstring123/replacement123/' firestore.rules` —
   real flat-deny by `path-escalate` (migration-tagged); file's md5
   confirmed unchanged.
3. `git status --short` — passed through uninspected, proving the widened
   matcher isn't a blanket deny.
4. `TARGET=firestore.rules; sed -i '' 's/a/b/' $TARGET` — real fail-closed
   deny: `path-escalate` refused to resolve the variable and denied rather
   than risk it.

**Residual, disclosed, not claimed closed**: a `cd` in a *prior, separate*
Bash tool call (not the same command string) is invisible to this hook —
it only ever sees one `tool_input.command` at a time. `sed -i` naming more
than one file only has its *last* token checked. Any mutation shape not in
the recognized list (a custom wrapper, an interpreter's own file-write
call) is as invisible as it always was. Full account in
`docs/tradeoffs.md`'s self-red-team, under the now-CLOSED `phase-gate`/
`path-escalate` entry (it documents what's closed and what's residual in
one place — read that, not just this summary).

**Also**: `docs/tradeoffs.md`'s residual-risks section now states the
deviations.md circuit-breaker's honor-system dependency plainly (it was
previously only in the self-red-team section) — no enforcement was added
for it, per the build prompt's own reasoning that mechanizing it would
mean intercepting model judgment, not a tool call.

### Fix 2 — Rescoped `lint`/`typecheck` to the changed-file set, added `floor --full`

**Before:** `lint`/`typecheck` were whole-tree, no-stdin capabilities. The
floor could not pass on any task in horizon because the tree already
carried 330 files of pre-existing formatting debt, independent of any
task's own diff.

**Decision (made outside this build, executed here): both paths, not
either.**

- `core/ADAPTER-CONTRACT.md §3` — `lint`/`typecheck` moved into the
  changed-file-set bucket (stdin, same convention as `test-changed`/
  `clone-scan`). New `§3.1` documents the rescoping and the `--full`
  escape hatch. `§4` (self-test convention) now requires a changed-file-
  set capability's `--self-test pass` fixture to include a sibling
  always-violating file *excluded* from the fed changed-set, proving
  scoping itself — not just that the tool wrapper works.
- `core/scripts/floor` — new `run_scoped` function, new `--full` flag.
  Default: `lint`/`typecheck` run scoped (stdin-fed changed files).
  `--full`: both run via `--full`, whole-tree, unscoped — for CI or a
  dedicated cleanup task. **Not wired into horizon's CI in this phase** —
  that's still future work, sequenced after the formatting-debt cleanup,
  same reasoning Phase D already gave.
- horizon's real `.spine/adapters/lint` and `.spine/adapters/typecheck`
  rewritten: `lint` scopes the *invocation* itself (dart format/eslint
  have no cross-file semantics); `typecheck` scopes the *verdict*
  (`flutter analyze` still needs whole-package context to resolve types
  correctly — only the pass/fail filter is scoped). Both gained `--full`.
- A fresh, minimal synthetic Python project (mypy/ruff only — Phase D's
  original 9-capability synthetic project no longer exists on disk;
  rebuilding all 9 was judged disproportionate to validating a
  2-capability contract change) built in scratch, with the same
  convention, to prove the rescoping generalizes beyond Dart/TS.

**Verified, not asserted** — the concrete regression test that matters
most: re-ran `floor 1 --task 20260808-fix-building-group-delete-orphans-units`
(the worked example's own task, previously the reason Phase D's headline
finding existed). It now reports `PASS: typecheck`, `PASS: lint`, and
correctly proceeds to fail at `test` for the separate, pre-existing,
already-disclosed reason (Very Good CLI counter-template suite / admin
`file_picker` version conflict) — not lint. The same invocation with
`--full` appended reproduces the original 330-file `lint: FAIL` exactly,
proving the escape hatch is a real whole-tree mode, not a no-op. The
synthetic Python project reproduces the same shape with a real, committed
`preexisting_debt.py`. `adapter-conformance --all` passes for both
capabilities in both projects, exercising the new scoping-proof self-test
fixtures (a sibling `other.py`/`other.dart`/`sample_other.js` present but
correctly out of scope).

**What this concedes** (see `docs/tradeoffs.md`'s "Phase E: the
lint/typecheck rescoping decision" for the full account): pre-existing
debt in untouched files becomes invisible to the per-task floor, by
design — the 330-file debt does not go away, it becomes something only
`--full`/CI will ever surface, and `--full` isn't wired anywhere yet.
`typecheck`'s verdict-filtering approach means a changed file's signature
edit that breaks an *unchanged* caller elsewhere is filtered out of the
scoped verdict — the same shape `test-changed`'s vacuous-pass-on-no-match
already carries.

### Fix 3 — Made silent tooling degradation loud

**Before:** the worked example's own task hit the Auto Mode classifier
wall (Phase D's top self-red-team finding) and silently hand-tracked phase
state with no ledger.json ever created — a self-concealing failure in the
exact mechanism meant to tell the engineer whether the system is worth
using. Nothing mechanical noticed.

**What changed:**
- `core/scripts/ledger` — new `note-gap <task-id> <script> <consequence>`
  subcommand (appends to a `tooling_gaps` array). `init`'s default JSON
  gained `hand_tracked:false` and `tooling_gaps:[]`. `aggregate` gained
  `tooling_gap_count` (sum across all tasks) and `hand_tracked_task_count`
  (tasks whose ledger.json had to be hand-authored because `ledger` itself
  was unreachable). Both jq expressions use `// []`/safe defaults so
  pre-Phase-E ledger.json files (missing these fields) don't break
  `aggregate`.
- `core/skills/task/SKILL.md` — new "tooling-gap discipline" section
  (three outcomes per script call: ran-passed / ran-failed / could-not-run;
  on could-not-run: append a `TOOLING GAP:` line to `work/<task-id>/
  notes.md`, call `ledger note-gap` if ledger itself is reachable, or
  hand-author a `hand_tracked:true` ledger.json stub if it isn't). Applied
  explicitly to `check-stale` and to `ledger init` itself.
- `core/skills/verify/SKILL.md` — same discipline applied to `floor`,
  `verdict-filter`, `conformance`, with specific defaults: floor
  could-not-run is reported as `FAIL` (never silently treated as pass),
  verdict-filter could-not-run falls back to the manual §5-schema check
  Phase D already improvised once, conformance could-not-run records
  exactly the consequence line the fix asked for verbatim ("conformance
  score unavailable — plan predictiveness unmeasured for this task").
  Step 5 assembly gained a "Tooling gaps" section, merged from
  notes.md plus anything hit during that `/verify` run.
- `core/skills/ship/SKILL.md` §3 — briefing.md now quotes verify.md's
  Tooling gaps section verbatim, same precedent as capability gaps.
- `core/templates/verify.md` / `core/templates/briefing.md` — new
  "Tooling gaps" sections.
- `core/skills/costs/SKILL.md` — reports `tooling_gap_count` and
  `hand_tracked_task_count` next to the bypass count, with the same
  "say so plainly if nonzero" framing.

**Verified, not asserted** — the actual open question Phase D flagged but
never confirmed: **does the wall affect an ordinary, attended session?**
Ran `ledger init/mark/set`, `check-stale`, `conformance`, and
`verdict-filter` for real, in this very session (attended — normal
permission flow, no `-p`, no `--dangerously-skip-permissions`), against
horizon's real worked-example artifacts (a scratch task ID for `ledger`,
a scratch copy of `research.md` for `check-stale` so its in-place
quarantine-banner write couldn't touch the real file, real `--out` paths
for `conformance`/`verdict-filter` reading the real `plan.md` and the real
`falsifier-verdict-raw.json`). **All four executed cleanly and wrote
exactly what they claim**: a real ledger.json with a real phase timestamp;
`check-stale` correctly detected that the worked example's own
`research.md` *is* stale (a grounding file changed after research was
written) and wrote the quarantine banner into the scratch copy; a real
precision/recall/f1 from `conformance`; 4/4 verdicts kept by
`verdict-filter`. **Confirms Phase D's suspicion directly**: the classifier
wall is specific to headless/unattended sessions, not the ordinary
attended case. Scratch task dir and scratch files cleaned up afterward;
horizon's real task artifacts were never touched (confirmed via `git
status` before/after).

**Deliberately not done**: the worked example's own real `verify.md`/
`briefing.md`/absent `ledger.json` were **not** retroactively rewritten to
the new template shape. Phase D's own stated principle for `docs/example/`
— "nothing was re-run or tidied to look more finished than the real run
was" — applies here too: that task's artifacts are a historical record of
what Phase D's tooling state actually produced, not a canvas for this
phase to polish. The new mechanism applies prospectively, to the next real
task run through the spine.

## 2. What a fresh session needs to know

- **All three fixes are narrowed or closed, not redesigned.** No new
  mechanism was added beyond what each fix explicitly asked for
  (`_bash-write-targets`, `floor --full`, `ledger note-gap`/
  `hand_tracked`). Nothing was promoted from the v2 shelf. The three
  recurring human touchpoints (class confirmation, plan approval, briefing
  read) are unchanged in kind — briefing gained one more section to read,
  not a new gate.
- **horizon's real worked-example task (`20260808-fix-building-group-
  delete-orphans-units`) was deliberately left exactly as this phase found
  it**: the 3-file bug fix is still real, correct, and uncommitted; the
  TOCTOU race is still unfixed and disclosed; the 330-file formatting debt
  is still unfixed. None of that was this phase's job — it's the
  engineer's first real task(s) through the system, per the build prompt's
  own instruction not to decide that unilaterally.
- **`floor --full` exists but is not wired anywhere.** It's a capability a
  human (or CI, once someone sets it up) has to invoke on purpose. Nothing
  runs it automatically. This is a real, disclosed gap, not an oversight —
  `docs/tradeoffs.md` says so explicitly.
- **The tooling-gap mechanism depends on the model actually following the
  recording discipline** — same honor-system caveat every other
  norm-not-hook-enforced mechanism in this system carries (the deviations
  circuit breaker has the identical shape). A skill that silently forgets
  to call `ledger note-gap` or write the `TOOLING GAP:` line recreates the
  exact silence this fix exists to close. Nothing mechanically forces the
  discipline; the fix makes following it cheap and visible when done, not
  unbypassable.
- **The Auto Mode classifier wall itself is still unfixed** — Phase E
  narrowed what it costs (confirmed scope, added visibility) but did not
  and could not fix the underlying classifier behavior, which is
  Anthropic-internal and outside what this build can see into.
- **Every hook branch touched this phase was watched firing** — four
  fresh `claude -p` subprocesses for Fix 1 (deny, flat-deny, pass-through,
  fail-closed-unresolved), all against horizon's real tree, all reported
  above and in `docs/tradeoffs.md`'s self-red-team.
- **`docs/tradeoffs.md` is the fuller record.** This handoff points at
  three new/updated sections there: "Phase E: the lint/typecheck rescoping
  decision," the updated "Auto Mode classifier wall" section ("Phase E:
  made the degradation loud"), and the updated self-red-team entries for
  `phase-gate`/`path-escalate` (now CLOSED, with residual shapes named)
  and the circuit breaker (moved into residual risks, stated plainly).
  Read those for the reasoning; this file is the punch list of what
  changed and what was verified.
