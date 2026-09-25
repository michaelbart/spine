---
name: autopilot
description: Experimental, unattended, whole-milestone-backlog execution — loops next-milestone-task, runs each task all the way to ship with every human stop self-resolved and logged (including Class 2's normally-mandatory ones), and ends with a single end-of-run report for after-the-fact review. Not a replacement for /task's normal flow. See docs/tradeoffs.md before using this.
disable-model-invocation: true
argument-hint: [--milestone <milestone-id>]
---

You are running `/autopilot`. `$ARGUMENTS` optionally carries
`--milestone <id>` to scope the whole run to one milestone instead of the
entire backlog. Scripts live at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`,
templates at `${CLAUDE_SKILL_DIR}/../../templates/<name>` — same
`${CLAUDE_SKILL_DIR}` expansion rule every other skill here uses: hand the
resulting path, `../../` included, to the shell verbatim; don't lexically
collapse it, `.claude/skills/autopilot` is a symlink and the shell has to
resolve `../../` against its real target.

## 0. What this is, in one paragraph

Spine's own autonomy dial already draws one line: "checks never scale down
with autonomy, only stops do" (`core/skills/task/SKILL.md`, the `autonomy`
field). `/autopilot` is that same dial pushed to its extreme point — every
remaining human stop `/task` has, including the ones normally treated as
structural rather than stylistic (Class 2's forced `guided` autonomy and
second-approver requirement), becomes "decide on your own judgment, log it,
keep going." What does **not** change: the deterministic floor, the
falsifier's mandatory stub-out probe, the security adversary,
`claims-check`, `conformance`, and `contract-touch` all still run for real
and can still fail a task outright. This removes stops, never checks. This
is a disclosed experiment (`docs/tradeoffs.md`),
not the default or recommended way to use spine — say so plainly if asked
about it, and never let it run silently disguised as an ordinary `/task`.

## 1. Preflight — real stops, not judgment calls

These are infrastructure/environment problems, not decisions autopilot's
own judgment has any business overriding — unlike everything in §4, these
remain genuine stops:

1. At least one `work/M*/milestone.md` must exist. If none do, say so and
   point at `/roadmap` (an already-scoped effort) or `/wayfinder` (a foggy
   one) instead of guessing at milestone boundaries — `/autopilot` consumes
   an already-planned backlog, it doesn't invent one. Stop here.
2. Core version check, identical to `core/skills/task/SKILL.md` Step 0:
   ```
   ${CLAUDE_SKILL_DIR}/../../scripts/setup --check --project <project root>
   ```
   `mismatch-strict` stops here, same message and same reason as `/task`'s
   own Step 0 — this machine's core is out of sync, no judgment call fixes
   that. `ok`/`unpinned`/`mismatch-warn` continue.
3. Git identity resolvable — `git config user.name` and `user.email` both
   non-empty, same requirement `/task`'s classify step already has. If
   either is empty, stop and say so exactly as `/task` does; never invent
   an identity.
4. If `.spine/current-task` already names an active task **and**
   `.spine/autopilot-active` does *not* exist, that task belongs to a human,
   not to a prior autopilot run — refuse to start, say plainly that a task
   is already active here and autopilot doesn't take over manual work, and
   stop. If `.spine/current-task` exists **and** `.spine/autopilot-active`
   also exists, this is a resumed autopilot run — skip to §3, continuing
   that task inline exactly where its own `state` file says, and log a
   `run-resumed` entry (§2) before proceeding.
5. Otherwise: write `.spine/autopilot-active` (one line, the start
   timestamp) and append a run-separator header to
   `.spine/autopilot-log.md` (create it with a one-line file header if it
   doesn't exist yet — this file accumulates across every `/autopilot`
   invocation, never truncated, so a prior run's history is never lost).

## 2. The log

Every override below appends one line to `.spine/autopilot-log.md`:

```
- <task-id> [<phase>] <kind>: <one-line what and why> — see <pointer>
```

`<kind>` is one of: `class-2-escalation`, `second-approver-override`,
`claims-check-override`, `halt-deviation-autoresolved`,
`circuit-breaker-reset`, `flag-autoacknowledged`, `new-milestone-confirmed`,
`run-resumed`, or `task-abandoned-unattended` (§5 — the one kind that means
a real failure, not a decision, and gets called out separately at the end).
`<pointer>` is whatever file a human would need to open to actually verify
the decision — `work/<task-id>/deviations.md#Deviation N`,
`work/<task-id>/approval.json`, `work/<task-id>/flags.json`, etc. This file
is the entire deliverable of the "review it after, not during" bargain —
treat every line as something a skeptical human will actually go check.

## 3. The loop

```
loop:
  ${CLAUDE_SKILL_DIR}/../../scripts/next-milestone-task --project <project root> \
    [--milestone <id> if §1's $ARGUMENTS carried one]
```

- **`TBD <id> <description>`** — don't prompt for confirmation the way
  `/task`'s own header does; treat it as confirmed, log a plain note isn't
  needed (this isn't an override, it's the expected normal case) and go to
  §4 with that description and milestone id.
- **`BLOCKED <id> <token>`** or **`WAITING <id>`** — nothing runnable right
  now. This is a structural sequencing fact, not a decision to render a
  verdict on — end the loop (§6), don't invent a way around it.
- **`NONE`** — every milestone complete, or none exist (shouldn't happen
  given §1.1, but if the last one just finished mid-run, this is the normal
  successful end). End the loop (§6).
- **Could not run at all** (the tooling-gap three-way distinction every
  skill here uses) — this is infrastructure failing, not a judgment call.
  Log it to `notes.md`-equivalent thinking (there's no active task folder
  to write into if this is between tasks) by noting it directly in the
  final report (§6), and end the loop rather than guessing what's next.

## 4. Running one task, inline, with every stop overridden

Follow `core/skills/task/SKILL.md` step by step, in this session, exactly
the way that skill's own §5 already directs `checkpointed`/`auto` autonomy
to follow `/verify`/`/ship` inline rather than through the Skill tool — same
technique, applied here to the whole of `/task`'s own procedure, not just
its tail end. **Nothing this run does leaves the machine — every
`registry-sync` call it makes (task/SKILL.md's own registry-sync calls,
`claims-check`, `propagate`, all of it) carries `--no-push`.** Every
commit stays real and local; nothing reaches a remote until a human
reviews the end-of-run report and decides to push it themselves. This is
by explicit instruction, not inferred, and it's the second way (alongside
skipping `open-pr`, below) this run's artifacts differ from ordinary
`auto` autonomy.

At classify time, write `work/<task-id>/autonomy` = `auto`
regardless of the computed class — this is `/autopilot`'s own ceiling
override, distinct from `/task`'s normal one where Class 2 forces `guided`.
**Every time the computed class is `2`** (by suggestion, protected-path
auto-escalation, or self-judgment), append a `class-2-escalation` log entry
naming what triggered it (the file/path, or the reasoning) before
continuing exactly as if it were Class 1 `auto` — this is the one override
explicitly called out for a dedicated end-of-run review, per the disclosed
tradeoff.

Apply these overrides at the exact point `task/SKILL.md`'s own text says to
stop or ask — everything else in that skill (research, plan-writing,
implementation, the floor, the falsifier, the security adversary,
`claims-check`, `conformance`, `contract-touch`) runs completely unchanged:

- **Plan approval** — already skipped by `auto` autonomy; nothing new here.
- **Class 2 second approver** (`task/SKILL.md` §3) — self-approve via the
  *existing* override path already defined there, never a new mechanism:
  `work/<task-id>/approval.json` with `"override": true,
  "override_reason": "autopilot: unattended run, no second approver
  available"`. Log a `second-approver-override` entry pointing at
  `approval.json`.
- **`claims-check` block** (`task/SKILL.md` §3) — take the "override"
  resolution `claims-check`'s own output already names as one of its three
  paths; append the `deviations.md` record that resolution already requires
  (tier `record-and-proceed`, since choosing to override *is* the
  resolution — identical to what a human choosing override would write).
  Log a `claims-check-override` entry naming the conflicting task, owner,
  and surface `claims-check` reported.
- **Halt-tier deviation** (`task/SKILL.md` §4) — do not stop. Decide using
  the plan's own "What I'll decide alone vs. stop and ask" section's
  context plus the best available judgment, exactly as if this were a
  `record-and-proceed` deviation instead — write the `deviations.md` record
  per `core/templates/deviations.md`'s shape with `- Tier: halt` (the tier
  stays honest — it really was a halt-tier decision point, only the
  *stopping* is overridden) but `- Status: resolved` immediately, and a
  `- Resolution:` line that opens `(autopilot, unattended):` followed by
  the real reasoning. If the resolution would normally require an
  amend-or-reaffirm on a `docs/charter.md` line, reaffirm it as-is (never
  silently edit the charter unattended) and record that. Log a
  `halt-deviation-autoresolved` entry quoting the record's own "Actually
  true" opening clause.
- **Circuit breaker** (`task/SKILL.md` §4, the third deviation on this
  task) — perform the mechanical reset exactly as written (`git stash push
  -u`, `state` back to `research`, bump `deviation_count`,
  `registry-sync`) but do not stop for a human: immediately re-delegate to
  the `researcher` agent and continue this same task through plan and
  implement again, inline. Log a `circuit-breaker-reset` entry. Track resets
  per task-id within this run; on the **second** reset for the same task
  (six total deviations on one task), do not attempt a third — go to §5
  instead.
- **Flag-blocked advance** (`task/SKILL.md`'s header note, every phase
  transition) — read the flag for real, decide for real whether the
  grounding it names still holds. Acknowledge it either way (`"acknowledged":
  true, "acknowledged_by": "autopilot"`, `"acknowledged_at"` set). If
  judged to still hold, log a `flag-autoacknowledged` entry with the
  reasoning. If judged not to hold, that's a real deviation — also apply
  the halt-tier row above (this compounds, it doesn't replace the
  acknowledgment).
- **`verify` FAIL** — `auto` autonomy already self-heals this (fix,
  re-verify inline, per `task/SKILL.md` §5) without stopping; nothing new
  here unless the fix itself hits a halt-tier deviation or the circuit
  breaker, which the two rows above already cover.
- **New-milestone-creation confirmation** (`task/SKILL.md`'s header,
  "no `milestone.md` yet") — confirm "proceed" itself rather than asking.
  Log a `new-milestone-confirmed` entry. Expected to be rare — §1.1 already
  requires at least one milestone to exist before autopilot starts at all —
  but a mid-run `TBD` entry can still name a milestone id with no file yet.
- **`/ship`'s commit and draft-PR steps** (`core/skills/ship/SKILL.md` §5/§5a)
  — commit locally exactly as §5 already writes (`git add -A -- <the task's
  actual changed paths, work/<task-id>/, docs/decisions/>`, then `git
  commit`), but **never `git push`** and never call the `open-pr` adapter,
  regardless of `.spine/profile.json`'s `pr_open` setting. Every task this
  run ships stays as a real, ordinary local commit on whatever branch is
  checked out — nothing reaches a remote until a human reviews the
  end-of-run report and pushes by hand. This is the one place this run's
  behavior differs from ordinary `auto` autonomy in a way that removes an
  *artifact* (and a network action), not a stop — the explicit choice made
  for this build, not a judgment call to re-litigate per task.
- **Worktree cleanup** (`ship/SKILL.md` §6, Extension D) — if this task
  happens to be running in a spine-created worktree, still ask the human at
  the end of `/autopilot`'s whole run whether to keep or remove it (fold
  it into §6's summary, don't ask mid-run) using this form (per `core/templates/human-touchpoint.md`):

  <!-- touchpoint:start -->
  > **Deciding:** whether to keep or remove the separate working copy this run used. It's yours because it's the only place the run's changes can be inspected before anything reaches a remote.
  > **Need to know:** The run finished in a temporary copy of the project at `<path>`. Nothing has been pushed.
  > **Recommend:** Keep it until you've reviewed the report — you can remove it afterwards.
  > 1. **Keep it** — next: I leave it in place; cost: some disk space; undo: yes, remove it later
  > 2. **Remove it** — next: I delete the copy; cost: you can't inspect it any more; undo: no
  > **Safe to ignore:** nothing.
  <!-- touchpoint:end -->

  A worktree left behind for a human to inspect is a feature here, not
  friction, given nothing else during the run gets reviewed until the end
  either.

## 5. Runaway guard — a real failure, not a decision

On a task's second circuit-breaker reset within this run (six total
deviations on one task, §4), stop retrying that task: leave `state` exactly
where it currently is — never force it to `done`, never fabricate a ship
(spine has no `abandoned` terminal state today, `docs/tradeoffs.md`
"Deferred (not built)"; inventing one unattended would be a bigger,
undisclosed decision, not a small one). Log a `task-abandoned-unattended`
entry — this kind is surfaced first and loudest in the end-of-run report,
never buried among ordinary overrides, because it's a real failure this
run couldn't resolve, not a confident decision. Continue the loop with
whatever `next-milestone-task` names next. If the abandoned task was
itself a later member task's blocking predecessor, the next
`next-milestone-task` call will correctly come back `BLOCKED` on it — that
is the loop ending correctly, not a bug to route around.

## 6. Ending the loop and the report

Whenever §3 ends the loop (`NONE`, `BLOCKED`, `WAITING`, or a tooling gap)
or a §1 stop fires: remove `.spine/autopilot-active` (a finished or
genuinely-stopped run is not "resumable" the way a mid-task interruption
is — a fresh `/autopilot` invocation re-derives everything from
`next-milestone-task` regardless). Then report, out loud, not just left as
a file:

- How the run ended, in plain words (milestone(s) complete; blocked on
  `<id>`/`<token>`; a tooling gap; or a preflight stop).
- Every task shipped this run — task id and title (the plan's own
  `# Plan: `<task-id>` — <title>` heading), in order.
- Every `task-abandoned-unattended` entry, named first and clearly marked
  as a real failure needing a look — never folded into the general count.
- Every other logged override, grouped by kind, with counts, and enough of
  a pointer per entry that a human can actually go verify each one — this
  is the single human touchpoint the whole run has, so it needs to be
  genuinely checkable, not a summary that just asserts things went fine.
- If any task ran in a spine-created worktree (Extension D), ask now
  whether to keep or remove each one (`core/skills/ship/SKILL.md` §6's
  own keep/remove choice, just deferred to here per §4's note above).

Point at `.spine/autopilot-log.md` for the complete, unsummarized list.
