---
name: task
description: Run the spine on a piece of work — classify, research, plan, approve, implement, verify, ship. The default way any non-trivial change gets made in a spine-installed project.
disable-model-invocation: true
argument-hint: [description of the work] [--milestone <milestone-id>]
---

**Output rule:** the `<!-- touchpoint:... -->` lines in this skill are lint markers; never print them. Show the human only the `>` lines between them.

You are running `/task`, the spine. `$ARGUMENTS` is the
task description as given, plus an optional `--milestone <milestone-id>`.

**When the task description in `$ARGUMENTS` is empty:** read `${CLAUDE_SKILL_DIR}/reference/milestone-and-auto-continue.md` and follow it exactly before continuing.

**When `$ARGUMENTS` contains `--milestone <id>`:** read `${CLAUDE_SKILL_DIR}/reference/milestone-and-auto-continue.md` and follow it exactly before continuing.

State lives in four places, and every phase transition below updates them
— they are not decoration, the hooks (`core/hooks/phase-gate`,
`path-escalate`) read them on every `Edit`/`Write`:

- `.spine/current-task` — the active task ID, one line. Absent = no active
  task = Class 0 default.
- `work/<task-id>/state` — the current phase name, one line.
- `work/<task-id>/class` — `0`, `1`, or `2`, one line.
- `work/<task-id>/autonomy` — `guided` or `auto`, one line
  (absent = `guided`, the default). This is the Phase 4 dial for *how many
  human stops* the flow has, orthogonal to `class` (which sets *how much
  verification*). **Blast radius caps autonomy, never the reverse:** a Class 2
  task is always `guided`; a Class 1 task may be `auto` or `guided`;
  Class 0 is `traced` (no task folder — §1). `/intake` sets this and enforces
  the ceiling; §3's escalation re-enforces it. `auto` runs with no scheduled
  stops — only the same tripwires every task has (a `halt`-tier deviation, a
  mid-stream escalation, the circuit breaker, a verify `FAIL`) pull the human
  in. Checks (floor, adversaries) never scale down with autonomy; only stops do.

**Resuming:** if `.spine/current-task` already exists, offer to resume that
task where it left off rather than starting a new one — do not silently
abandon it. Read the existing task's `state` and `class`, and re-derive
nothing from memory; a fresh session has none, so read
`work/<task-id>/{research,plan,deviations}.md` before proceeding. If a new
description was typed while a task is already active, don't swallow it into
the resume offer: ask plainly whether to resume `<existing-task-id>` (phase
`<state>`) or abandon it and start `<new description>`, and follow the human's answer;
never silently pick one. To abandon, ask for a one-line reason and run
`${CLAUDE_SKILL_DIR}/../../scripts/set-state <existing-task-id> abandoned "<reason>"`
(the folder stays on disk; it clears `.spine/current-task`).

**Step 0 — install health check.** Before classifying, run
`$(readlink -f "${CLAUDE_SKILL_DIR}")/../../scripts/setup --check --project <project root>`
(`realpath` if `readlink -f` is missing). No output: continue. A missing or stale
`.claude/hook-guard`: show it, continue, and say edits are not gated until `setup`
is re-run. Exit 1 (invalid `.spine/profile.json`): stop and show it. If `setup`
could not run, note a tooling gap and proceed; an unreachable check is not a pass.

Scripts referenced below live at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`.
Templates live at `${CLAUDE_SKILL_DIR}/../../templates/<name>`.
`${CLAUDE_SKILL_DIR}` is a placeholder you expand to this skill's own
directory; hand the resulting path — including the `../../` — to the shell
verbatim. Do **not** lexically collapse `skills/task/../..` to `.claude/`:
`.claude/skills/task` is a symlink into the spine core checkout, so the shell
must resolve `../../` against the symlink's real target (`<spine>/core/...`).
Collapsing it as text yields a nonexistent `.claude/scripts/...` path and a
"no such file" error.

**When a core script could not run at all (blocked, denied, or errored before its own logic executed):** read `${CLAUDE_SKILL_DIR}/reference/plan-implement-edge-cases.md` and follow it exactly before continuing.


**Voice.** Every message this skill leaves for the human follows
`core/templates/human-touchpoint.md`. Before sending one, run its "Before you
send" list: gloss or drop internal names, IDs and commit hashes, keep one
decision per message, and put surprises first.

## 1. Classify — the first recurring human touchpoint

**When the file `.spine/current-intake` exists:** read `${CLAUDE_SKILL_DIR}/reference/intake-handoff-and-ticket-branch.md` and follow it exactly before continuing.

Every task gets a class, and the human confirms it — not the model alone.
Suggest one, don't decide it unilaterally (for a direct `/task` you also
confirm an autonomy right after the class — see "Autonomy for a direct
`/task`" below; `/intake` instead proposes it from a code-grounded pre-scan):

- **Class 0 (trivial):** suggest when the change looks like it will touch
  ≤2 files, ≈15 lines or fewer (or this project's `.spine/profile.json`
  `class0_max_files`/`class0_max_lines` if set), introduces no new public
  symbol, and (check against `.spine/protected-paths.conf`) touches no
  protected path. No work folder, no phases — but not invisible: make the
  edit, run the cheap check `${CLAUDE_SKILL_DIR}/../../scripts/floor 0 --project <project root>`
  (types and lint on the changed files only; it logs its result), then commit it carrying a `Spine-Ticket: <key>` trailer if a ticket
  key is derivable from the branch name (split on `-`, match against
  `.spine/ticket-pattern.conf` if present). If no ticket is derivable
  (genuinely off-ticket), make the edit and skip the trailer; spine doesn't
  chase off-ticket one-offs — they stay visible via the org's own commit
  convention. The backstop is `path-escalate`: with no active task it
  defaults to class 0, so if the edit turns out to touch a protected path,
  the hook halts it and you tell the human plainly: "this stopped being
  trivial" — then restart as a real task. The same goes if `floor 0` fails and
  the fix is no longer small. If it reports DEGRADED (no type or lint check is
  set up here), proceed, but say so in one line.
- **Class 1 (standard):** the default for anything bigger than that.
- **Class 2 (high blast radius):** suggest when the human's description or
  your own quick read implies protected-path or schema/contract/auth
  changes. Entry requires the human's explicit confirmation — never infer
  your way into Class 2 silently. (It can also be triggered automatically
  later, at plan time, if the predicted-touch list turns out to intersect a
  protected path — see step 3.)

Ask the class with `AskUserQuestion`, in this form (per
`core/templates/human-touchpoint.md`) when you're confident; when you're torn
between classes, use the full block `/intake` uses for that case (its "Torn or
low confidence" form). Name the class in plain words ("a small edit, no
ceremony" / "a standard change" / "a high-risk change: checked with you at
every step"), not by number, unless glossed:

<!-- touchpoint:start confirm -->
> **Run this as a <trivial | standard | high-risk> change?** <One sentence: what it touches, and what happens automatically if that reading turns out wrong.>
> **Yes** (recommended) — <what happens>. **No, <the alternative in plain words>** — <what that costs you>.
<!-- touchpoint:end -->

**Autonomy for a direct `/task`** (no intake handoff; the choice only exists at
Class 1 — Class 0 is `traced`, Class 2 is always `guided`, the ceiling). Once the
class is confirmed, ask the human how autonomous the flow should run: `guided`
(stop at each phase — the default and the safe choice) or `auto` (no scheduled
stops; you review the finished PR). Default to `guided` if they express no preference. **If you had
suggested a higher class than the human chose** (e.g. you suggested Class 2, they
picked Class 1), lean `guided` and say why — a change you read as
higher-blast-radius is exactly the kind to keep a human in the loop on, even at
the class they chose. Cap the offer at `.spine/profile.json`'s `autonomy_ceiling`
if set (`guided|auto`; a legacy `checkpointed` ceiling is read as `guided`). You'll write the result to `work/<task-id>/autonomy` in the setup step
below, the same place the class file is written. (`/intake` proposes this from
its pre-scan; a direct `/task` does none, so it simply asks.) Ask it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start -->
> **Deciding:** how often I should stop and check with you on this task. It's yours because it trades your attention against how closely you watch the work.
> **Need to know:** The two settings are `guided` (I stop after every phase and wait for you) and `auto` (I run without scheduled stops; you review the finished PR).
> **Recommend:** The setting called guided — it is the safe default; pick less stopping only if the change is small and easy to review.
> 1. **Stop after every phase** — next: I pause after research, plan and each step for your go-ahead; cost: the most of your attention; undo: yes, you can switch later
> 2. **No scheduled stops** — next: I run everything and you review the finished PR; cost: you find problems late; undo: yes, nothing merges without you
> **Safe to ignore:** the setting names; I'll use plain words.
<!-- touchpoint:end -->

Once confirmed, for Class 1/2: generate the task ID
`<YYYYMMDD>-<kebab-slug>` (today's date, a short slug from the description),
`mkdir -p work/<task-id>`, write `.spine/current-task`, write
`work/<task-id>/class`, then log it:
`${CLAUDE_SKILL_DIR}/../../scripts/spine-event class-set class=<0|1|2> autonomy=<mode>`
(local event log; never fails, prints nothing). **If `--milestone <id>` was given, do this next
step now, before writing `work/<task-id>/state`** — write
`work/<task-id>/milestone` = `<id>` and replace this milestone's first
still-`TBD` member-task entry with `<task-id>` in `work/<id>/milestone.md`,
per this skill's own header note above (`phase-gate` only gates
`Edit`/`Write` once a `state` file exists and reads `research`/`plan`; no
`state` file yet means this edit is a genuine no-op for the hook to
evaluate, not an exemption it has to special-case). **Refresh the project map
if it needs it — this is automatic, never a question for the human.** Run
`${CLAUDE_SKILL_DIR}/../../scripts/map-age --project <project root>`. On
`fresh`, do nothing. On `stale <n>`, `empty` or `missing`, spawn a
`general-purpose` agent (Agent tool) told to follow
`core/skills/remap/SKILL.md`'s steps and write `docs/map.md` (`/remap` itself
can't be invoked by a skill, and it must run now, before `state` exists, because
once `state` reads `research` the `phase-gate` hook only lets writes into
`work/<task-id>/`). Say one plain line, "Refreshing the project map (about a
minute)", and log the outcome:
`${CLAUDE_SKILL_DIR}/../../scripts/spine-event map-refresh result=<ok|failed> was=<stale n|empty|missing>`.
If the refresh fails or times out, carry on: the researcher reads the code
directly, exactly as it did before, and nothing is reported to the human.
`/ship` commits the refreshed map on its own. Only after that: `${CLAUDE_SKILL_DIR}/../../scripts/set-state <task-id> research`.

## 2. Research

Delegate to the `researcher` agent (Agent tool, `subagent_type: researcher`)
with a delegation message containing: the task description, the task ID, the
class, whether this is a bug fix (root-cause mode) or not, and pointers to
`docs/charter.md` / `docs/map.md` if they exist.

**Class 1 is research-lite:** tell the researcher explicitly to ground only
the files the change will directly touch or directly call into, and skip a
wider subsystem survey unless the change's own blast radius forces it (e.g.
it touches a symbol the charter or an existing `docs/decisions/` entry
already flags as widely shared). **Class 2 is full research:** no such
limit — survey the actual subsystem and its real callers. This line is a
real design decision; defend or revise it in `docs/tradeoffs.md`, don't
silently drift from it task to task.

**When the task's description or likely touch set includes a path declared in `.spine/ui-paths.conf` or `.spine/ui-content-paths.conf`:** read `${CLAUDE_SKILL_DIR}/reference/ui-path-steps.md` and follow it exactly before continuing.
The researcher's entire reply is the complete `research.md` content
(including its header) — write it verbatim to `work/<task-id>/research.md`.

## 3. Plan

Run `${CLAUDE_SKILL_DIR}/../../scripts/set-state <task-id> plan`. Then run
`${CLAUDE_SKILL_DIR}/../../scripts/check-stale work/<task-id>/research.md`
— if it reports stale, the grounding drifted since it was written; regenerate
research (back to step 2) before planning on it — **unless every drifted item
is this task's own bookkeeping** (false-positive rule, next paragraph).
Whether to redo research is spine's call, never the human's: don't ask. If `check-stale` could not
run at all (see the tooling-gap discipline above), do not treat that as
"assume fresh" — record the gap ("research staleness unmeasured for this
task") and proceed on the assumption research *might* be stale, noting that
explicitly when you present the plan for approval so the human's review
accounts for it.

**When `check-stale` reports drift and the only drifted item is `work/<id>/milestone.md` for this task's own milestone:** read `${CLAUDE_SKILL_DIR}/reference/plan-implement-edge-cases.md` and follow it exactly before continuing.

Write `work/<task-id>/plan.md` yourself, following
`${CLAUDE_SKILL_DIR}/../../templates/plan.md`'s structure exactly, and
its prose sections (the gist, what could go wrong, how we'll know it
worked) per the writing mandate at
`${CLAUDE_SKILL_DIR}/../../templates/writing-mandate.md` — plain
language, bottom line first, nothing pushed below a section's first
line. The `## Predicted touch` section (inside its own `<!-- MACHINE:
predicted-touch -->` fence) is machine-parsed verbatim by
`core/scripts/conformance`, don't reformat it.
**200-line hard cap, comments included** — `wc -l < plan.md` it before
presenting; if it doesn't fit, the task splits into two, it does not get
compressed into unreadability. If
this task belongs to a milestone (`work/<task-id>/milestone` set), check that milestone's `## Known gaps for
future member tasks`: if this plan's approach actually closes one of its
`gap-<n>` entries, add the `## Resolves known gaps` section per
`core/templates/plan.md` naming it — this is what lets `/ship` §3c remove
the entry once the task ships; a gap this plan resolves without
citing it stays listed, which is a missed cleanup, not a wrong one, so
don't invent a citation just to clear the section. If any decision from
`/design` grounds this plan, add the `## Grounds on decisions` section per
`core/templates/plan.md`.

**When `work/<task-id>/milestone` is unset (no `--milestone` was given):** read `${CLAUDE_SKILL_DIR}/reference/plan-implement-edge-cases.md` and follow it exactly before continuing.

Check every `## Predicted touch` entry against `.spine/protected-paths.conf`. If any match and `work/<task-id>/class` is not
already `2`, auto-escalate: rewrite the class file to `2`, **and
rewrite `work/<task-id>/autonomy` = `guided`** (the ceiling — a Class 2 task
is never `auto`; escalation pulls the human back in), log it
(`${CLAUDE_SKILL_DIR}/../../scripts/spine-event class-escalated from=<old> to=2
when=plan`), and
say so plainly when you present the plan — this is plan-triggered
escalation; it does not need a separate confirmation prompt beyond the
plan approval you're about to ask for anyway.

**When the plan's predicted touch includes a path declared in `.spine/ui-paths.conf` or `.spine/ui-content-paths.conf`:** read `${CLAUDE_SKILL_DIR}/reference/ui-path-steps.md` and follow it exactly before continuing.

**When `work/<task-id>/autonomy` is `auto`:** read `${CLAUDE_SKILL_DIR}/reference/auto-autonomy-steps.md` and follow it exactly before continuing.

**Present the plan and stop — this is the second recurring human
touchpoint.** Do not proceed to implementation in the same turn. Wait for
explicit approval. If the human requests changes, revise and re-present;
this doesn't count against the deviation circuit breaker, it's pre-approval
iteration. Present it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start -->
> **Deciding:** whether I may start building this plan. It's yours because nothing changes until you approve, and after that I edit real files.
> **Need to know:** <the plan's bottom line in one plain sentence>. <If research staleness went unmeasured: "One caveat: I couldn't check whether my research notes are still current.">
> **Recommend:** <Approve | Revise> — <one-line reason from the plan's own risks>
> 1. **Approve** — next: I start building; cost: none; undo: yes, git can revert everything until it ships
> 2. **Ask for changes** — next: you tell me what to change and I show the plan again; cost: a few minutes; undo: n/a
> **Safe to ignore:** the file lists and internal labels in `plan.md`; the summary above is the whole decision.
<!-- touchpoint:end -->

**Record the approval** — `work/<task-id>/approval.json`:
`{"approver": "<git identity>", "at": "<iso8601>"}`. The approver is whoever's
session this is, resolved from `git config user.name`/`user.email` in *this*
session. Log it with
`${CLAUDE_SKILL_DIR}/../../scripts/spine-event plan-approved autonomy=<mode>`. This holds at every
class: the person who owns the task approves its plan. Class 2 adds more
checking (guided at every step, protected-path escalation, the adversaries),
not a second approver.

## 4. Implement

On approval: run
`${CLAUDE_SKILL_DIR}/../../scripts/set-state <task-id> implement`. This is what unblocks
`phase-gate` — it only restricts writes during `research`/`plan`.

Work the plan's steps directly (you have full tool access again; `phase-gate`
no longer applies, `path-escalate`/`dep-gate` still do). For each decision
you hit, **first ask whether it's a setup event, not a deviation at all**:
**When a decision comes up during implementation that does not match the plan:** read `${CLAUDE_SKILL_DIR}/reference/plan-implement-edge-cases.md` and follow it exactly before continuing.
Whenever the class is rewritten mid-implementation, log it too:
`${CLAUDE_SKILL_DIR}/../../scripts/spine-event class-escalated from=<old> to=2
when=implement`.

For everything that *is* a real deviation, match it against the plan's
`## What I'll decide alone vs. stop and ask` section — its three lists
carry the same fixed tier keywords `deviations.md`'s own `- Tier:` field
uses (`decide-alone` / `record-and-proceed` / `halt`), so the match is
literal, not judgment-call vocabulary translation:

- **I'll just do** (`decide-alone`) — just decide, keep going, no record.
- **I'll do and note** (`record-and-proceed`) — append a record to
  `work/<task-id>/deviations.md` (use
  `${CLAUDE_SKILL_DIR}/../../templates/deviations.md`'s shape; tier
  `record-and-proceed`, status `resolved` immediately since proceeding *is*
  the resolution), then keep going.
**When a decision matches the plan's `halt` list (`I'll stop and ask before`):** read `${CLAUDE_SKILL_DIR}/reference/plan-implement-edge-cases.md` and follow it exactly before continuing.

Log each one as you record it, so a later recap can compare what was logged
with what the diff shows:
`${CLAUDE_SKILL_DIR}/../../scripts/spine-event deviation kind=real tier=<record-and-proceed|halt>`
for a `deviations.md` record, and `... spine-event deviation kind=setup` for a
`SETUP:` note.

**When this task's third `deviations.md` record is being written, or a resolution cites a `docs/charter.md` line:** read `${CLAUDE_SKILL_DIR}/reference/plan-implement-edge-cases.md` and follow it exactly before continuing.

## 5. Verify and ship

Implementation acceptance
checks (from the plan) should already pass before you move on — check them
yourself first; don't hand a known-broken diff to `/verify`. Then: run
`${CLAUDE_SKILL_DIR}/../../scripts/set-state <task-id> verify`.

**This phase's shape depends on `work/<task-id>/autonomy`** (absent =
`guided`). The independence that `/verify` protects comes from the falsifier
and security agents being *fresh, isolated subagents* — never from who typed
the command — so `auto` preserves it while removing the human
relay the engineer explicitly delegated by choosing that autonomy at `/intake`.

**guided** — `/verify` and `/ship` both carry `disable-model-invocation: true`,
so this skill cannot call either via the Skill tool; only the human literally
typing `/verify <task-id>` (or `/ship <task-id>`) gets through. Tell the human
plainly: implementation is ready, please run `/verify <task-id>` yourself. Then
stop and wait — this session does not proceed to ship on an unverified diff, the
same waiting posture step 3 uses at plan approval. When resumed,
**read `work/<task-id>/verify.md` directly** (its `Result:` line reads `PASS` or
`FAIL` verbatim). If `FAIL`: fix it (back to implementation, same task) and ask
the human to re-run `/verify`. If `PASS`: run `${CLAUDE_SKILL_DIR}/../../scripts/set-state <task-id> ship`,
then ask the human to run `/ship <task-id>` and wait the same way. Say
it in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start short -->
> **What happened:** the work is built and ready to be checked.
> **What it means for you:** I can't start the check myself; only you typing the command can, so I'm paused and nothing more happens until you do.
> **To continue:** type `/verify <task-id>`.
<!-- touchpoint:end -->

When you report that implementation is done (and again after `/ship`
completes), use this form; it replaces any freeform summary. Show only the
`>` lines, never the `<!-- touchpoint:... -->` marker lines:

<!-- touchpoint:start report -->
> **Bottom line:** <where things stand in one plain sentence, and whether anything waits on you>
> **What I did:** <2-4 short lines: what now works or behaves differently, in user terms, not file names>
> **What you need to do:** <the next step and exact command, or "nothing">
> **Worth knowing:** <anything surprising, left open on purpose, or not proven, in plain words; or "nothing">
<!-- touchpoint:end -->

**When `work/<task-id>/autonomy` is `auto`:** read `${CLAUDE_SKILL_DIR}/reference/auto-autonomy-steps.md` and follow it exactly before continuing.

For every mode, `/ship` (however it runs) handles the merge gate, the commit
trailer(s), the briefing, and clearing `.spine/current-task`.

When resumed after `/ship`, confirm it actually completed by checking that
`.spine/current-task` no longer names this task (`/ship` clears it on
success) and that `work/<task-id>/briefing.md` exists, rather than taking
the human's word alone.

**Point the human at the delta briefing path when `/ship` completes** — that
read is the third recurring touchpoint, and it happens once, at the end,
not as a gate you enforce mid-flow.

