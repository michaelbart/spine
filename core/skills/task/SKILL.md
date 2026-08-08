---
name: task
description: Run the spine on a piece of work — classify, research, plan, approve, implement, verify, ship. The default way any non-trivial change gets made in a spine-installed project.
disable-model-invocation: true
argument-hint: [description of the work]
---

You are running `/task`, the spine (build prompt §2.2). `$ARGUMENTS` is the
task description as given; if empty, ask for one before doing anything else.

State lives in three places, and every phase transition below updates them
— they are not decoration, the hooks (`core/hooks/phase-gate`,
`path-escalate`) read them on every `Edit`/`Write`:

- `.spine/current-task` — the active task ID, one line. Absent = no active
  task = Class 0 default.
- `work/<task-id>/state` — the current phase name, one line.
- `work/<task-id>/class` — `0`, `1`, or `2`, one line.

**Resuming:** if `.spine/current-task` already exists, read its `state` and
`class` and offer to resume that task where it left off rather than starting
a new one — do not silently abandon it. Re-derive nothing from memory; a
fresh session has none, so read `work/<task-id>/{research,plan,deviations}.md`
before proceeding.

Scripts referenced below live at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`.
Templates live at `${CLAUDE_SKILL_DIR}/../../templates/<name>`.

## 1. Classify — the first recurring human touchpoint

Every task gets a class, and the human confirms it — not the model alone
(build prompt §2.7). Suggest one, don't decide it unilaterally:

- **Class 0 (trivial):** suggest when the change looks like it will touch
  ≤2 files, ≈15 lines or fewer, introduces no new public symbol, and (check
  against `.spine/protected-paths.conf`) touches no protected path. No work
  folder, no ledger entry, no phases — just make the edit. The backstop is
  `path-escalate`: with no active task it defaults to class 0, so if the
  edit turns out to touch a protected path, the hook halts it and you tell
  the human plainly: "this stopped being trivial" — then restart as a real
  task. This threshold is an initial value (build prompt open question
  §5.2); `/costs` data is what should revise it, not intuition.
- **Class 1 (standard):** the default for anything bigger than that.
- **Class 2 (high blast radius):** suggest when the human's description or
  your own quick read implies protected-path or schema/contract/auth
  changes. Entry requires the human's explicit confirmation — never infer
  your way into Class 2 silently. (It can also be triggered automatically
  later, at plan time, if the predicted-touch list turns out to intersect a
  protected path — see step 3.)

Once confirmed, for Class 1/2: generate the task ID
`<YYYYMMDD>-<kebab-slug>` (today's date, a short slug from the description),
`mkdir -p work/<task-id>`, write `.spine/current-task`, write
`work/<task-id>/class`, write `work/<task-id>/state` = `research`, then
`${CLAUDE_SKILL_DIR}/../../scripts/ledger init <task-id>` and
`ledger mark <task-id> classify`.

## 2. Research

`ledger mark <task-id> research`. Delegate to the `researcher` agent (Agent
tool, `subagent_type: researcher`) with a delegation message containing: the
task description, the task ID, the class, whether this is a bug fix
(root-cause mode) or not, and pointers to `docs/charter.md` / `docs/map.md`
if they exist.

**Class 1 is research-lite:** tell the researcher explicitly to ground only
the files the change will directly touch or directly call into, and skip a
wider subsystem survey unless the change's own blast radius forces it (e.g.
it touches a symbol the charter or an existing `docs/decisions/` entry
already flags as widely shared). **Class 2 is full research:** no such
limit — survey the actual subsystem and its real callers. This line is the
crisp version of build prompt open question §5.5; defend or revise it in
`docs/tradeoffs.md`, don't silently drift from it task to task.

The researcher's entire reply is the complete `research.md` content
(including its header) — write it verbatim to `work/<task-id>/research.md`.
Harvest its subagent transcript into the ledger under phase key
`research-agent` (see §6). Then harvest the main session's own
`classify`→`research` window into phase key `research`.

## 3. Plan

`ledger mark <task-id> plan`, write `state` = `plan`. First run
`${CLAUDE_SKILL_DIR}/../../scripts/check-stale work/<task-id>/research.md`
— if it reports stale, the grounding drifted since it was written; regenerate
research (back to step 2) before planning on it.

Write `work/<task-id>/plan.md` yourself, following
`${CLAUDE_SKILL_DIR}/../../templates/plan.md`'s structure exactly — the
`## Predicted touch` section is machine-parsed verbatim by
`core/scripts/conformance`, don't reformat it. **200-line hard cap,
comments included** — `wc -l` it before presenting; if it doesn't fit, the
task splits into two, it does not get compressed into unreadability.

Check every `## Predicted touch` entry against
`.spine/protected-paths.conf`. If any match and `work/<task-id>/class` is
not already `2`, auto-escalate: rewrite the class file to `2`, and say so
plainly when you present the plan — this is the plan-triggered escalation
build prompt §2.2 describes; it does not need a separate confirmation
prompt beyond the plan approval you're about to ask for anyway.

**Present the plan and stop — this is the second recurring human
touchpoint.** Do not proceed to implementation in the same turn. Wait for
explicit approval. If the human requests changes, revise and re-present;
this doesn't count against the deviation circuit breaker, it's pre-approval
iteration.

## 4. Implement

On approval: `ledger mark <task-id> implement`, write `state` = `implement`.
This is what unblocks `phase-gate` — it only restricts writes during
`research`/`plan`.

Work the plan's steps directly (you have full tool access again; `phase-gate`
no longer applies, `path-escalate`/`dep-gate` still do). For each decision
you hit, use the plan's latitude table:

- **Decide-alone** — just decide, keep going, no record.
- **Record-and-proceed** — append a record to `work/<task-id>/deviations.md`
  (use `${CLAUDE_SKILL_DIR}/../../templates/deviations.md`'s shape; tier
  `record-and-proceed`, status `resolved` immediately since proceeding *is*
  the resolution), then keep going.
- **Halt** — schema, public contracts, new dependencies, auth logic, or
  anything protected-path (the hooks enforce the file-level cases
  independently). Append a deviations.md record with status `open`, stop
  implementing, and tell the human what you need resolved. This is a
  legitimate non-recurring touchpoint (build prompt §2.7) — it does not
  happen on every task, only when reality diverges from the plan in a
  halt-tier way.

**Circuit breaker:** count every deviations.md record regardless of tier.
On the third for this task, the plan is invalidated — `git stash push -u -m
"spine: circuit breaker, work/<task-id>"` to preserve what you'd built
without losing it, write `state` back to `research`, bump
`work/<task-id>/ledger.json`'s `deviation_count` (see §6), and tell the
human plainly: three wrong guesses means the research was wrong once, not
that each guess should be patched forward. Fresh research is required
before re-planning.

If a resolution (halt or otherwise) cites a `docs/charter.md` line, it must
end amend-or-reaffirm: the human either edits that charter line or
reaffirms it as-is, dated, and the deviations.md resolution records which.

## 5. Verify and ship

Implementation acceptance checks (from the plan) should already pass before
you move on — check them yourself first; don't hand a known-broken diff to
`/verify`. Then: `ledger mark <task-id> verify`, write `state` = `verify`,
invoke `/verify` (Skill tool) with the task ID.

If `/verify` reports the floor failed: fix it (back to implementation,
same task, not a new deviation by itself unless the fix itself diverges
from the plan) and re-run `/verify`. If it passed: write `state` = `ship`,
invoke `/ship` (Skill tool) with the task ID. `/ship` handles the merge
gate, the commit trailer, the briefing, and clearing `.spine/current-task`.

**Point the human at the delta briefing path when `/ship` completes** — that
read is the third recurring touchpoint, and it happens once, at the end,
not as a gate you enforce mid-flow.

## 6. Ledger bookkeeping (every phase transition above)

Session transcript path: `$HOME/.claude/projects/$(pwd | tr '/' '-')/${CLAUDE_SESSION_ID}.jsonl`.
**Rule: whichever `ledger mark <task-id> <phaseN+1>` call you make is also
what harvests the phase it's closing out** —
`ledger harvest <task-id> <phaseN> --transcript <path> --from <phaseN's
stored mark timestamp> --to <the phaseN+1 timestamp you just wrote>`. Read
phaseN's stored timestamp back from `work/<task-id>/ledger.json`, don't
recompute it. Concretely: marking `research` harvests `classify`; marking
`plan` harvests `research`; marking `implement` harvests `plan`; marking
`verify` harvests `implement`. `/ship` continues the same rule for the
phases after this skill hands off (see `core/skills/ship/SKILL.md` §4) and
additionally harvests its own final `ship` window, since nothing marks a
phase after it.

For a subagent (researcher, falsifier, security), harvest its own
transcript (`.../subagents/agent-<id>.jsonl`, `<id>` from the Agent tool's
result) in full under its own phase key (`research-agent`, `falsifier`,
`security`) — this is separate bookkeeping from the window-based harvest
above, not a replacement for it; the orchestrating phase's own window still
covers the main session's overhead around the delegation.

`ledger set <task-id> class_declared <class>` at step 1;
`ledger set <task-id> class_escalated true` if step 3's auto-escalation
fired; `ledger set <task-id> deviation_count <n>` whenever it changes.
