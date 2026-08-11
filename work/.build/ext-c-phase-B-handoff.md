# Extension C, Phase B — portable install, version pin, ownership/registry, `/tasks`

Folds in the Phase A review: version pinning/skew detection (2.1
addition), decision-store-exists (not A-installed) as `propagate`'s real
branch condition, sibling files as the actual registry with an enforced
flag-block, and the classifier-wall doc committed (`91e041b`) before this
phase started.

## Built

| File | Status |
|---|---|
| `core/scripts/setup` | New. Per-machine install/repair + pin init/compare. Real, executable, tested. |
| `core/scripts/registry-sync` | New. add/commit/push + pull-rebase-retry for one task's `work/<task-id>/`, scoped, never `-A`. |
| `core/templates/claims.json` | New. `_comment`-documented schema, same convention `workspace.json`'s template already uses. |
| `core/skills/tasks/SKILL.md` | New. Read-only registry view — pull, enumerate non-`done` tasks, report owner/class/phase/claims/flags. |
| `core/skills/task/SKILL.md` | Extended: step 0 (cheap pin check), registry sibling-file init at classify, flag-blocked-advance check + `registry-sync` at every phase transition, claims.json real population at plan. |
| `core/skills/bootstrap/SKILL.md` §5 | Extended: delegates symlink/settings wiring to `setup` instead of hand-symlinking + hardcoding an absolute `permissions.allow` entry. |
| `core/skills/workspace/SKILL.md` §2 | Extended: same delegation — the workspace root had the identical bug, not called out in the build prompt's manifest but clearly the same class of fix. |
| `README.md` | Extended: second-engineer join flow, version-pin/upgrade-workflow section under "Staying installed." |
| `work/.build/ext-c-phase-A-handoff.md` | Committed at session start (`91e041b`), together with the prior build's 513-line Extension A/B doc section — see that commit for why both landed together (user-approved). |

Not yet committed (deliberately — Phase D is where this build's own
commit happens, matching the prior extension build's own per-extension
commit granularity, not per-sub-phase).

## Verified firing

**Reproduced the break, then proved the fix, on a genuinely different
machine** (not a same-machine path relocation — confirmed in Phase A that
doesn't actually break the absolute path). Docker container, user `bob`,
`/home/bob`, spine's own tree extracted to `/opt/spine-clone` (a path that
shares nothing with `/Users/michaelbart/spine`), `bgr`'s real (pre-
migration) git history cloned in:

```
BEFORE: test -e .claude/hooks → HOOKS_BROKEN
$ /opt/spine-clone/core/scripts/setup --project /home/bob/proj/bgr
setup: untracked 14 pre-existing committed symlink path(s) — commit this to finish the migration
setup: initialized .../core-pin.json at 91e041b... (mode: warn)
setup: core pin ok
setup: ok — hooks/skills/agents/rules resolve to /opt/spine-clone
AFTER: test -e .claude/hooks → HOOKS_OK, readlink → /opt/spine-clone/core/hooks
```

Then fired a real hook through the repaired symlink, not just checked it
resolves: `phase-gate` correctly denied a write outside the task folder
during `research` (exit 2, real denial message) and allowed one inside it
(exit 0) — the repaired install is not just present, it's functionally
correct. Re-ran `setup` a second time: no-op, exit 0, confirming
idempotence. `git status --short` after: only the expected untrack +
`.gitignore` + `.spine/core-pin.json` changes — no manual path surgery
anywhere in the sequence.

**Skew detection, both modes, real invocations**, against the real
(non-Docker) `~/bgr` install: hand-set the pin to a fake sha under
`mode: strict` → `setup --check` printed the skew message and exited 1;
same fake sha under `mode: warn` → printed the warning, exited 0; restored
the real sha → `ok`, exit 0. All three via `--check` (read-only, no
symlink/pin writes), matching what `/task`'s new step 0 actually calls.

**Registry sharing, real git push/pull, two separate checkouts of a
common bare remote (no shared filesystem)** — this is the same standard
Phase D's real demo needs, run now to de-risk the mechanism before wiring
`claims-check`/`propagate` on top of it in Phase C:
- Engineer A opens a real task (`mkdir work/<id>`, writes `class`,
  `state`, `owner` from real `git config`, `claims.json` from the real
  template, `flags.json = []`), `registry-sync` commits and pushes.
  Engineer B, on an independent clone, pulls and enumerates open tasks
  with the exact snippet `/tasks` §2–3 documents — sees Engineer A's task,
  correct owner string, correct class, correct phase, `0/0` flags,
  **having learned of it only through the pushed registry state**.
- Two engineers pushing concurrently to different task folders: second
  push rejected, `registry-sync` auto pull-rebased and retried, succeeded
  — real fast-forward-then-retry, not simulated.
- Two engineers writing the *same* task's `flags.json` without pulling
  first (the genuine-conflict case, open question 6): push rejected,
  rebase hit a real `CONFLICT (add/add)`, `registry-sync` refused to guess
  and exited 1 with "resolve by hand" — this is the answer to open
  question 6's "confirm a conflict... resolves sanely": sanely means
  *fails loud, leaves git's own conflict markers, never silently drops
  one side*, not silent auto-merge.
- Flag-block detection logic (hand-written flag entry, not yet produced by
  a real `propagate` — that's Phase C): `jq` query for
  `acknowledged==false` correctly found it and produced the exact
  human-readable block message `task/SKILL.md` §"Flag check" prose
  describes (what changed, when, by whom, which grounding entry).

## Decisions and disagreements (with the plan, or with the build prompt's literal text)

- **Manifest scope, exceeded on purpose.** The deliverable manifest didn't
  list `core/skills/bootstrap/SKILL.md`, `core/skills/workspace/SKILL.md`,
  or `core/scripts/registry-sync`. Touched anyway: 2.1's own acceptance
  criterion ("a second checkout... runs the throwaway-hook test and the
  full floor with zero manual path surgery") is unreachable without
  changing what `bootstrap`/`adopt`/`workspace` actually write at install
  time — the manifest's file list reads as illustrative, not exhaustive,
  and I'm treating it that way rather than leaving a known-identical bug
  in `workspace`'s own install step. `registry-sync` exists because four
  call sites (`/task`'s own five write points, `claims-check` and
  `propagate` in Phase C, `/ship`'s Phase C extension) need byte-identical
  add/commit/push/retry logic — spine's own convention is "one script owns
  one piece of logic," not five copies of the same git incantation in five
  skills' prose.
- **Shared mainline = whatever branch the engineer is already on, not a
  dedicated `work/`-only branch.** The build prompt names both as
  workable (§2.2). Rejected the dedicated-branch option for a concrete,
  primitive-level reason, not preference: `phase-gate`/`path-escalate`/
  `dep-gate` (out of scope to modify, confirmed in Phase A, not in the
  manifest) read `work/<task-id>/state` etc. as plain files relative to
  `$CLAUDE_PROJECT_DIR` — the actual working tree, no git-ref indirection.
  A separate branch's task-folder commits would be invisible to those
  hooks unless the engineer's working tree *was* that branch, which
  contradicts "still working on their own code." Committing registry
  writes straight onto the current branch is therefore the only choice
  compatible with "no hook changes," not just the simpler one — recorded
  here so Phase D's tradeoffs-doc pass states this as a primitive
  constraint, not a stylistic pick.
- **`setup` also strips a stale absolute-path `Bash(...)` entry from the
  *committed* `.claude/settings.json`** on every run (added mid-phase,
  after the first real `bgr` test showed the old entry surviving
  untouched). Conservative rule: only entries whose pattern starts
  `Bash(/` are stripped; `.spine/adapters/*`-shaped entries are always
  left alone. Confirmed against real `bgr` output — settings.json now
  carries only the two relative adapter patterns.
- **`--check` never touches the pin file, even to initialize it.** An
  unpinned project reports `unpinned` under `--check`, not an error — this
  matters for a project mid-`/bootstrap` or one that adopted spine before
  Extension C existed and hasn't run plain `setup` yet; `/task`'s step 0
  should not be what silently creates a pin, only a real `setup` run
  should.

## What Phase C needs from this handoff

- `registry-sync` is the call every new Phase C mechanism should reuse,
  never reimplement: `claims-check` doesn't write anything itself (it only
  reads other tasks' `claims.json`), but `propagate` writes `flags.json`
  entries into *other* tasks' folders and must `registry-sync <other-task-
  id>` after each write — a flag that stays local until that other
  engineer's next pull isn't a flag, per build prompt §2.5's own framing.
- The flag-block *check* already lives in `task/SKILL.md`; Phase C's job
  is the *writer* (`propagate`) plus a real demonstration of the refusal
  actually firing end-to-end (a real flag written by one engineer's ship,
  a real block on the other engineer's next phase-advance attempt) — the
  Phase B test above only proved the detection query works on a
  hand-written flag, which is necessary but not sufficient evidence per
  the review's "same standard as a hook" requirement.
- `propagate`'s decision-grounding branch: per the accepted correction,
  branch on `docs/decisions/` existing and being non-empty, never on
  Extension A being installed — `check-stale` already establishes this is
  baseline infrastructure (Phase A §0.4).
- `core/skills/ship/SKILL.md`'s own final `registry-sync` (closing
  `state` = `done`) and the second-approver field both still need
  writing — untouched in Phase B on purpose, since they're explicitly
  Phase C's line items and `/ship` wasn't touched here.
- Ledger `engineer` field and `/costs` per-engineer view: also untouched,
  per the original plan's phase split — `ledger init`'s jq skeleton and
  `ledger aggregate`'s reduce are both straightforward extensions, not
  blocked on anything built in this phase.
- Real fixtures already proven live in this phase, reusable for Phase C
  and the Phase D demo without rebuilding them: the `registry-test` bare-
  remote + two-clone scratch setup (`/private/tmp/.../scratchpad/ext-c/
  registry-test`, ephemeral — Phase D's real demo should recreate this
  shape against real checkouts, not reuse the scratch paths directly) and
  `~/bgr` itself, now migrated (uncommitted — see below).

## `~/bgr` — real changes sitting uncommitted, for your review

Running `setup` against the real `~/bgr` (necessary to prove the fix
against a real historical install, not just a synthetic one) left real,
uncommitted changes there: 14 symlinks untracked, `.gitignore` gained 5
lines, `.claude/settings.json` lost its absolute-path allow entry,
`.claude/settings.local.json` gained the machine-local equivalent, and
`.spine/core-pin.json` was created (pinned to this spine checkout's current
`HEAD`). Left uncommitted per "only commit when asked" — same precedent as
the original build leaving `horizon`'s `callers.md` fix uncommitted for the
engineer's own review. Nothing in `~/bgr`'s own project files (adapters,
capabilities, protected paths, real code) was touched.
