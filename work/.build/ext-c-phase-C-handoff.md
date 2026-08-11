# Extension C, Phase C — claims-check, propagate, ship-time re-grounding, second-approver, ledger attribution

Opened with the carried-over verification demand (fresh-clone enforcement
state) before this phase's own build — that finding changed Phase C's
scope (added `.claude/hook-guard`) and is folded in below rather than
repeated. `~/bgr`'s migration landed as `7b9f7bc` before this phase's
build work started; nothing here assumed the old uncommitted state.

## Built

| File | Status |
|---|---|
| `core/templates/hook-guard` | New (carried in from the verification demand). Committed-to-every-project stub that fails closed when the real hook is absent. |
| `core/scripts/setup` | Extended: writes/refreshes `.claude/hook-guard`, rewires `settings.json`'s `PreToolUse` commands through it. |
| `core/skills/bootstrap/SKILL.md` | Extended: initial `settings.json` write routes through the guard from the first commit. |
| `core/scripts/claims-check` | New. Write/write + write/read block, read/read + shared-decision/contract warn. Pulls the shared mainline first. |
| `core/scripts/propagate` | New. Flags every other open task grounding on a changed file/decision/contract; severity from the target task's own phase (advisory research/plan, disruptive implement+); `registry-sync`s each flag it writes. |
| `core/skills/task/SKILL.md` | Extended: `approval.json` recording at plan approval (self for Class 0/1, real-second-approver-or-loud-override for Class 2). |
| `core/skills/ship/SKILL.md` | Extended: §0 ship-time re-grounding (check-stale + post-rebase floor, both ledger-recorded); §1 second-approver gate; §5 `propagate` invocation on merge; §6 final `registry-sync`; briefing sections for both. |
| `core/scripts/ledger` | Extended: `engineer` field at `init` (from `git config`, same resolution as `owner`); `aggregate --by-engineer`. |
| `core/skills/costs/SKILL.md` | Extended: per-engineer view, framed explicitly as an instrument, not a leaderboard. |

## Verified firing

**Fresh-clone enforcement state (carried-over demand) — empirical, via
your `!` passthrough since this session's own Bash tool can't invoke
`claude` (the documented wall):**
```
BEFORE fix: claude -p "Write canary.txt" in a fresh clone with settings.json
  wiring hooks that don't exist yet → canary.txt CREATED. Fail-open,
  confirmed, not assumed.
AFTER fix (hook-guard committed, settings.json routed through it):
  same test → "The write was blocked by a local hook guard: ..." →
  canary2.txt absent. Fail-closed, confirmed.
```
Mitigation is in `core/templates/hook-guard` + `setup`'s rewiring, both
committed to spine's own tree and to `~/bgr` (`7b9f7bc`).

**`claims-check` asymmetry, real cases, all four shapes exercised in one
scratch project (`work/.build/` scripts, not narrated):**
```
task-1 predicted_touch=[auth.ts, login.ts], grounds on session.ts
task-2 predicted_touch=[auth.ts, session.ts], grounds on shared.ts
task-3 predicted_touch=[other.ts], grounds on shared.ts

claims-check task-1 → BLOCKED:
  write/write: both predict touching 'auth.ts'
  write/read:  task-2 predicts 'session.ts', task-1's research grounds on it
  (+ 1 warning: shared decision grounding D-1 — not blocking)
  exit 1

claims-check task-3 → clear, 1 warning (read/read on shared.ts with
  task-2 — no write conflict), exit 0

Renegotiation: task-2's predicted_touch revised (auth.ts dropped),
claims-check task-1 re-run → clear, exit 0.
```
Read/read never blocks; write/write and write/read always did, in the
same run, against real files — this is the asymmetry the build prompt
says is the whole reason the check is usable in a real codebase.

**Flag-blocked phase advance, full loop, two independent git checkouts of
one bare remote, no shared filesystem — same standard as a hook firing,
per the review's own bar:**
```
Engineer A opens task-x (research phase), grounds on src/lib/foo.ts,
  registry-syncs (real push).
Engineer B ships task-y, changes src/lib/foo.ts, runs
  `propagate task-y --by "Engineer B ..."` with the changed path on
  stdin → "propagate: flagged 20260811-task-x (1 entry, severity=advisory)"
  → registry-sync pushes the flag.
Engineer A `git pull --rebase` → flags.json now shows Engineer B's flag,
  acknowledged:false, having learned of it only through the pushed
  registry.
Flag check (task/SKILL.md's own rule) → "REFUSED — 1 unacknowledged
  flag(s): file 'src/lib/foo.ts' changed (task 20260811-task-y shipped)
  (by Engineer B ..., severity: advisory, ...)" — state stays 'research'.
Engineer A acknowledges (edits flags.json, registry-syncs) → re-check →
  "CLEAR — advancing" → state written to 'plan', registry-synced.
```

**`propagate` branches on `docs/decisions/` existing, never on Extension A
being installed** — confirmed by construction, not just by the review's
own correction: the script's decision-matching branch only ever runs
against whatever `--decisions` lists and whatever a target task's
`claims.json.grounding_decisions` contains — nothing in it checks for
`/design`, `docs/decisions/DEFERRED.md`, or any other A-specific file. A
project with zero decisions ever recorded (A absent or simply unused)
sees this branch produce zero matches, the same way an empty
`--contracts` list produces zero matches on a single-repo (B-absent)
project — one mechanism, degrading by absence of data, not by a presence
check.

**Ship-time re-grounding, both halves real:** `check-stale` re-run after
simulating a neighbor's merge landing (a real second commit changing the
grounding file) correctly flipped from `ok` to `STALE`, and
`ledger set ship_time_regrounding "check-stale: stale"` recorded it. The
post-rebase floor re-run reuses `core/scripts/floor` verbatim — already
proven extensively in the original build's own worked examples — Phase C
only added the *call site* (`/ship` §0) and the ledger recording around
it, not new floor logic; not independently re-verified here since nothing
about `floor` itself changed.

**Second-approver, all four real branches**, jq logic matching `/ship`
§1's own prose exactly:
```
no approval.json               → HALT: no approval.json
approver == owner, no override → HALT: approver == owner, no override
approver == owner, override    → PROCEED (loud) — reason shown verbatim
approver != owner               → PROCEED (real second approver: <id>)
```

**`ledger aggregate --by-engineer`**, real invocation against two real
`ledger.json` files (`engineer` field populated at `init` from
`git config`, one per simulated engineer): produced a correct per-engineer
breakdown (`task_count`, `class_escalation_count`, `bypass_count`,
`tooling_gap_count`, `claims_conflicts`, `avg_deviation_count`), grouped
correctly by the `engineer` string.

## Decisions and disagreements

- **`approval.json` is a new sibling file**, not something folded into
  `claims.json` or `state`. Same reasoning as Phase A's owner/claims/flags
  split — a second approver is a distinct fact from declared surfaces,
  and cramming it in would make `claims-check`'s own file harder to reason
  about for no benefit.
- **Second-approver enforcement lives partly in `task/SKILL.md` (recording,
  at plan approval) and partly in `ship/SKILL.md` (gating, at merge)** —
  deliberately split, not duplicated: `/task`'s own session can never
  self-assert a second identity (it can only ever resolve its own `git
  config`), so it can only *ask for* a real second approval or record an
  explicit override; `/ship` is what actually verifies the recorded fact
  before merging. Demonstrated as jq logic matching the prose exactly
  (above) rather than as a new script — this check has exactly one call
  site (`/ship` §1), unlike the git add/commit/push sequence
  `registry-sync` exists to deduplicate across five-plus call sites.
- **`propagate`'s severity field answers open question 2 mechanically**:
  the target task's own `state` at write time, not a separate
  classification a human or the source task has to supply. Simple,
  derived from data already on hand, no new input required at any call
  site.
- **`ledger.json`'s new fields** (`claims_conflicts`, `ship_time_regrounding`,
  `second_approver`) are written by `set`, same convention as every
  existing field — no new ledger subcommand needed. `claims-check` itself
  never calls `ledger set` (it's a standalone script deliberately, runnable
  outside a task-lifecycle context too, per the build prompt) — the
  invocation *was* missing from `/task`'s own plan-approval step on a
  first pass (claims.json got populated but `claims-check` was never
  actually called to gate anything, silently defeating §2.3's whole
  point). Caught while writing this handoff, not by a separate review —
  fixed in `task/SKILL.md` before this phase closed: claims-check now
  runs before the plan is presented, blocks presentation on an unresolved
  conflict, and records `claims_conflicts` on every blocking run
  (override or not, since a conflict that gets resolved by waiting still
  happened and should count).

## What Phase D needs from this handoff

- Every mechanism above has now been proven individually, live, but never
  all together in one continuous run against two *real* project checkouts
  (not scratch fixtures invented per-mechanism) — that composition is
  Phase D's actual job, not a re-derivation of any single piece.
- `~/bgr` is the natural B-absent fixture (already migrated, already has
  the hook-guard fix) for one half of Phase D's solo-regression proof;
  a workspace-shaped fixture (this build's own `spine/` repo has no
  `workspace.json` itself, so a genuine multi-repo test bed would need
  the same `/workspace` init flow this build's predecessor already
  exercised against `bookmarks`/`bookmarks-cli` — not rebuilt here, worth
  reusing if still present, or explicitly noted absent if not).
- All scratch fixtures used in this phase's tests live under
  `/private/tmp/.../scratchpad/ext-c/` and are ephemeral — Phase D should
  build its own real checkouts for the two-engineer demo per the build
  prompt's own requirement ("two checkouts, two git identities... a
  container for the second machine"), not reuse these paths directly.
