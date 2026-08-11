---
name: task
description: Run the spine on a piece of work — classify, research, plan, approve, implement, verify, ship. The default way any non-trivial change gets made in a spine-installed project.
disable-model-invocation: true
argument-hint: [description of the work] [--milestone <milestone-id>]
---

You are running `/task`, the spine (build prompt §2.2). `$ARGUMENTS` is the
task description as given, plus an optional `--milestone <milestone-id>`;
if the description is empty, ask for one before doing anything else.

**Step 0 — core version check (Extension C §2.1, the "cheap session-start
check").** No SessionStart-shaped hook exists to carry this — the
manifest forbids a new one — so it lives here, the one recurring entry
point every real task passes through:

```
${CLAUDE_SKILL_DIR}/../../scripts/setup --check --project <project root>
```

`ok`/`unpinned`: continue. `mismatch-warn`: show the warning, continue —
this machine's core may enforce differently than what this project was
calibrated against, but it's not a halt. `mismatch-strict`: stop here,
show the message, do not classify or touch any task state until the
engineer has pulled this machine's spine checkout to the pinned sha or a
maintainer has bumped the pin (`docs/tradeoffs.md`'s Extension C section
has the full upgrade workflow). If `setup` itself could not run at all,
this is the tooling-gap discipline below's "could not run" case —
note it and proceed, don't treat an unreachable check as a passing one.

**Multi-repo (Extension B)**: if `workspace.json` exists at the project
root, this session's own project root *is* the workspace root, and this
one `/task` invocation is the single task folder, single plan, single
human approval for however many member repos the change touches (build
prompt §2 — never a separate `/task` per repo). Every step below runs
exactly once, at the workspace root; the only things that change shape are
the `## Predicted touch` list (repo-qualified) and the plan-time escalation
check in §3 — both called out inline below. **A project with no
workspace.json runs every step below exactly as it always has** — this is
the zero-behavioral-change guarantee at the skill level.

**If `--milestone <id>` is given:** read `work/<id>/milestone.md` (design-
stage extension, `core/templates/milestone.md`) before classifying — its
`## Member tasks` list, `## Inter-task contracts` (what a prior member task
in this milestone left true, which this task may assume without
re-verifying), and `## Capability targets` all become planning context for
every phase below. Once this task's ID is generated (step 1), replace this
milestone's first still-`TBD` member-task line with the real task ID
(`Edit` on `work/<id>/milestone.md` — this is bookkeeping, not a phase
artifact, so it's exempt from `phase-gate`'s task-folder restriction the
same way any pre-approved administrative edit would need to be; do it
during step 1, before `phase-gate` would even apply). Record
`work/<task-id>/milestone` = `<id>`, one line, so `/ship` (final member
task's own done-definition check) and a resumed session both know this
task belongs to a milestone without re-parsing `$ARGUMENTS`.

State lives in three places, and every phase transition below updates them
— they are not decoration, the hooks (`core/hooks/phase-gate`,
`path-escalate`) read them on every `Edit`/`Write`:

- `.spine/current-task` — the active task ID, one line. Absent = no active
  task = Class 0 default.
- `work/<task-id>/state` — the current phase name, one line.
- `work/<task-id>/class` — `0`, `1`, or `2`, one line.

**The registry (Extension C §2.2/§2.3), sibling files alongside the three
above — never crammed into `state` itself**, which every hook and skill
above already reads as a bare one-line phase name
(`work/.build/ext-c-phase-A-handoff.md`'s own recorded reason for this
split):

- `work/<task-id>/owner` — one line, the git identity
  (`git config user.name <user.email>`) that created this task. Written
  once, at classify, never edited.
- `work/<task-id>/claims.json` — this task's declared surfaces
  (`core/templates/claims.json`). Empty skeleton at classify, populated
  for real at plan approval (§3).
- `work/<task-id>/flags.json` — array, `[]` at classify. Written by
  `core/scripts/propagate` (Extension C §2.5) when another task's ship
  changes something this task grounds on.

**Flag-blocked advance — the real enforcement point, not a suggestion.**
Before writing `state` forward at *any* of the three phase transitions
below (research→plan, plan→implement, implement→verify), read
`work/<task-id>/flags.json` first. If any entry has `"acknowledged":
false`, refuse to advance: tell the human exactly what changed, when, by
whom, and which grounding entry it hit (the flag's own fields — quote
them, don't paraphrase), and stop. The human resolves it the same way any
halt-tier deviation resolves — by acting on the information, then editing
that flag entry (`"acknowledged": true`, `"acknowledged_at"`,
`"acknowledged_by"` set) — never by silently clearing it or advancing
around it. This is the mechanism `core/scripts/propagate`'s flags exist to
be *for*; a flag nothing ever reads back would be exactly the "manufactures
confidence" failure shape build prompt §1 names for a hook that doesn't
fire — `/task` is what performs every phase transition, so `/task` is what
owns this check, the same way `phase-gate` owns write-restriction during
research/plan.

**Registry sync — every write to any file in this list, and every phase
transition, is followed by:**

```
${CLAUDE_SKILL_DIR}/../../scripts/registry-sync <task-id> --project <project root> --message "<short reason>"
```

This is what makes "the registry is shared state" (Extension C §2.2) real
rather than aspirational — a task folder that only exists on its creator's
own machine is invisible to `claims-check` and `propagate` running
anywhere else. `registry-sync` stages only `work/<task-id>/`, never the
rest of the working tree (mid-implement code edits sitting elsewhere stay
untouched and uncommitted, exactly as `phase-gate`'s own restriction
already implies they should during research/plan). A "no remote
configured" or "nothing changed" result is a normal, silent success, not a
tooling gap — only a real push failure after `registry-sync`'s own
rebase-retry is.

**Resuming:** if `.spine/current-task` already exists, read its `state` and
`class` and offer to resume that task where it left off rather than starting
a new one — do not silently abandon it. Re-derive nothing from memory; a
fresh session has none, so read `work/<task-id>/{research,plan,deviations}.md`
before proceeding.

Scripts referenced below live at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`.
Templates live at `${CLAUDE_SKILL_DIR}/../../templates/<name>`.

**Tooling-gap discipline (applies to every script invocation below, not
just ledger):** every time you invoke a core script, distinguish three
outcomes, not two — "ran and passed," "ran and failed" (a real result, act
on it normally), and **could not run at all** (the tool call itself was
blocked, denied, or errored before the script's own logic ever executed —
a permission denial, a sandbox/classifier block, "command not found" from
a broken symlink; not the script exiting non-zero on its own). The third
state is the one that's silently indistinguishable from the second if you
don't name it — see `docs/tradeoffs.md`'s Auto Mode classifier wall finding
for why this matters. On "could not run":

1. Append a line to `work/<task-id>/notes.md` (create it, header `# Notes`,
   if it doesn't exist yet): `TOOLING GAP: <script> could not run — <one-line
   consequence>.` Be concrete about the consequence (e.g. "research
   staleness unmeasured for this task," not "check-stale failed").
2. If `ledger` itself is reachable (this gap is about some *other* script),
   also run `ledger note-gap <task-id> <script> "<consequence>"` — this is
   what feeds `/costs`' tooling-degradation count mechanically instead of
   leaving it as prose only a human reading notes.md would find.
3. If `ledger` itself is what's unreachable — including `ledger init` never
   having succeeded for this task — hand-author `work/<task-id>/ledger.json`
   directly (Write tool) with the same shape `ledger init` would have
   produced (see `core/scripts/ledger`'s `init` case for the exact fields)
   but with **`hand_tracked: true`** and a `tooling_gaps` array containing
   at least this gap. This is the stamp that keeps the task from being
   silently invisible to `ledger aggregate` (which globs
   `work/*/ledger.json`) — a real ledger.json with `hand_tracked:false` and
   a hand-authored one with `hand_tracked:true` are visually and
   mechanically distinguishable to anyone reading either the file or
   `/costs`' output.
4. Never let a could-not-run script silently read as "nothing to report."
   Degrade gracefully (keep going by hand, per the phase's own fallback —
   e.g. hand-tracking `state` below) but the degradation itself must leave
   a trace in at least one of notes.md / ledger.json / verify.md.

This discipline is what `/verify` (step 1 below hands off to) and `/ship`
carry forward into `verify.md`'s and `briefing.md`'s own "Tooling gaps"
sections — notes.md is this task's running log until then.

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
`ledger mark <task-id> classify`. If `ledger init` could not run at all,
this is the ledger-itself-unreachable case in the tooling-gap discipline
above — hand-author the `ledger.json` stub (`hand_tracked: true`) right
now, at task creation, rather than waiting for a later phase to notice; a
task that never gets a ledger.json at all is invisible to `/costs`, not
just degraded. **If `--milestone <id>` was given**, also write
`work/<task-id>/milestone` = `<id>` and replace this milestone's first
still-`TBD` member-task entry with `<task-id>` in `work/<id>/milestone.md`
now, per this skill's own header note.

**Registry init (Extension C §2.2), same step, before the first
`registry-sync`:** resolve owner identity —
`git config user.name` and `git config user.email`. **If either is empty,
stop before creating the task folder** and tell the human plainly: "spine's
ownership model reads git identity, it does not invent one — run `git
config --global user.name '<you>'` and `--global user.email
'<you@example.com>'` first." (Primitive verification,
`ext-c-phase-A-handoff.md` §0.2 — a fresh machine genuinely has neither set;
never fall back to `$USER`, hostname, or any other guess.) Otherwise write
`work/<task-id>/owner` = `<name> <email>`, one line. Write
`work/<task-id>/claims.json` from `core/templates/claims.json` with
`task_id` filled in and every array empty (populated for real at plan
approval, §3). Write `work/<task-id>/flags.json` = `[]`. Then:

```
${CLAUDE_SKILL_DIR}/../../scripts/registry-sync <task-id> --project <project root> --message "task: open <task-id>"
```

This is the literal "committed and pushed to it at task creation" build
prompt §2.2 requires — a task invisible to a colleague's `claims-check`
until `/ship` would defeat the entire mechanism, so this happens now, not
deferred to the end of the phase.

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

**Flag check first** (per this skill's own header note on flag-blocked
advance): read `work/<task-id>/flags.json`; any unacknowledged entry halts
here, before anything else in this step. `ledger mark <task-id> plan`,
write `state` = `plan`, `registry-sync <task-id>`. First run
`${CLAUDE_SKILL_DIR}/../../scripts/check-stale work/<task-id>/research.md`
— if it reports stale, the grounding drifted since it was written; regenerate
research (back to step 2) before planning on it. If `check-stale` could not
run at all (see the tooling-gap discipline above), do not treat that as
"assume fresh" — record the gap ("research staleness unmeasured for this
task") and proceed on the assumption research *might* be stale, noting that
explicitly when you present the plan for approval so the human's review
accounts for it.

Write `work/<task-id>/plan.md` yourself, following
`${CLAUDE_SKILL_DIR}/../../templates/plan.md`'s structure exactly, and
its prose sections (the gist, what could go wrong, how we'll know it
worked) per the writing mandate at
`${CLAUDE_SKILL_DIR}/../../templates/writing-mandate.md` — plain
language, bottom line first, nothing pushed below a section's first
line. The `## Predicted touch` section (inside its own `<!-- MACHINE:
predicted-touch -->` fence) is machine-parsed verbatim by
`core/scripts/conformance`, don't reformat it (multi-repo: every entry
repo-qualified, `<repo-name>:<path>`, per the template's own comment).
**200-line hard cap, comments included** — `wc -l` it before presenting;
if it doesn't fit, the task splits into two, it does not get compressed
into unreadability. Multi-repo, additionally: write `## Ship order` the
moment `## Predicted touch` names more than one repo. Write `## Contract
change` if research or your own reading of `## Predicted touch` suggests
this plan touches a declared contract's producer paths or spec — this is a
judgment call at plan time, since `core/scripts/contract-touch` itself
needs a real diff and can't run yet; `/verify` (step 5 below) runs it for
real against the actual diff regardless and fails the task if a `breaking`
classification and this line disagree, so a wrong guess here is caught,
never silently trusted. See `core/templates/plan.md`'s own comments and
`core/rules/contracts.md` for what each value means. If
this task belongs to a milestone (`work/<task-id>/milestone` set), the
plan's `## The gist` must be consistent with that milestone's `##
Inter-task contracts` — what a prior member task already left true is a
real constraint on this plan, not optional context; if the plan needs to
violate one, that's a deviation against the milestone itself and belongs
in the gist's own rejected-alternative reasoning, said explicitly, not
silently contradicted. If any decision from `/design` grounds this plan,
add the `## Grounds on decisions` section per `core/templates/plan.md`.

Check every `## Predicted touch` entry against `.spine/protected-paths.conf`
— single-repo, that's always this project's own file. **Multi-repo: check
each entry against its *own* repo's `.spine/protected-paths.conf`**
(strip the `<repo-name>:` prefix, resolve the repo's absolute path via
`workspace.json`, read that repo's own conf) — checking every entry across
every repo in one pass is what makes this "escalate if *any* repo's
protected path is hit" loop the mechanical form of build prompt §2's "class
escalation composes as max across repos": there is no separate max
computation to write, it falls out of checking every entry regardless of
which repo it belongs to. If any match and `work/<task-id>/class` is not
already `2`, auto-escalate: rewrite the class file to `2` (one file, at the
workspace root for a multi-repo task — one class for the whole task), and
say so plainly when you present the plan — this is the plan-triggered
escalation build prompt §2.2 describes; it does not need a separate
confirmation prompt beyond the plan approval you're about to ask for
anyway.

**Populate `work/<task-id>/claims.json` for real** (Extension C §2.2/§2.3),
now that a plan exists: `predicted_touch` from `## Predicted touch`
verbatim, `grounding_files`/`grounding_decisions` from `research.md`'s own
header, `contracts` from `## Contract change`'s named contract if present,
`updated_at` set to the current timestamp. `registry-sync <task-id>` — this is the version of claims.json a
colleague's `claims-check` (Phase C) sees; a claims.json still at its
empty classify-time skeleton would make every intersection check
vacuously pass, silently defeating the whole mechanism.

**Run `claims-check` before presenting the plan** (Extension C §2.3 — "invoked
by the plan-approval step of `/task`"):

```
${CLAUDE_SKILL_DIR}/../../scripts/claims-check <task-id> --project <project root>
```

Print any warnings (read/read or shared grounding — informational, never
blocking). If it blocks (exit 1): **do not present the plan for approval
yet** — show the human the specific conflicting task(s), owner(s), and
surface(s), and the three resolution paths verbatim from its own output
(wait / renegotiate scope / override). Renegotiate means revising `##
Predicted touch` and re-running this check, same as any other plan
revision. Override means proceeding anyway, loudly: append a note to
*this* task's `deviations.md` (tier `record-and-proceed`, since choosing
to override is itself the resolution) naming the conflicting task, and
`ledger set <task-id> claims_conflicts <n>` (`n` = the blocking count
`claims-check` reported) so `/costs`' per-engineer view has real data
instead of a permanent zero — do this on *every* blocking run, override
or not, since a conflict that gets resolved by waiting/renegotiating
still happened and is worth counting. If clear (exit 0, warnings or not),
proceed straight to presenting the plan.

**Present the plan and stop — this is the second recurring human
touchpoint.** Do not proceed to implementation in the same turn. Wait for
explicit approval. If the human requests changes, revise and re-present;
this doesn't count against the deviation circuit breaker, it's pre-approval
iteration.

**Record the approval** (Extension C §2.6) — `work/<task-id>/approval.json`:
`{"approver": "<git identity>", "at": "<iso8601>", "override": false,
"override_reason": null}`. `registry-sync <task-id>`.

- **Class 0/1**: the approver is whoever's session this is — resolve from
  `git config user.name`/`user.email` in *this* session, same as `owner`.
  Self-approval is expected and correct here; build prompt §2.6 is explicit
  that mandatory cross-review does not extend to Class 1 — "recreates the
  review-bottleneck theater spine exists to escape."
- **Class 2**: the approver must be a *different* git identity than
  `work/<task-id>/owner`. This session cannot manufacture that identity —
  it can only ever resolve its own `git config`. So: if this session's own
  identity equals `owner`, **do not write `approver` as this session's own
  identity and call it approved.** Tell the human plainly: "Class 2 needs a
  second approver. Have a colleague pull this project, read
  `work/<task-id>/plan.md` (already on the shared mainline — that's what
  makes this possible without a separate review tool), and if they
  approve, run their own session and write
  `work/<task-id>/approval.json` themselves (their own `git config`
  identity, not typed/asserted) + `registry-sync`." Then stop — this
  session waits (pull periodically, or the human says when it's done)
  rather than proceeding to implement on an unapproved Class 2 plan.
  **Override** (a genuine solo/vacation-coverage situation, same trust
  model as `/ship --bypass`): the owner may self-approve by writing
  `approval.json` with `"override": true` and a real
  `"override_reason"` — loud, not silent; `/ship` (Phase C's own
  extension) surfaces this in the briefing and the ledger unconditionally,
  never treats it as an ordinary approval.

## 4. Implement

**Flag check first**, same rule as step 3. On approval: `ledger mark
<task-id> implement`, write `state` = `implement`, `registry-sync
<task-id>`. This is what unblocks `phase-gate` — it only restricts writes
during `research`/`plan`.

Work the plan's steps directly (you have full tool access again; `phase-gate`
no longer applies, `path-escalate`/`dep-gate` still do). For each decision
you hit, match it against the plan's `## What I'll decide alone vs. stop
and ask` section — its three lists carry the same fixed tier keywords
`deviations.md`'s own `- Tier:` field uses (`decide-alone` /
`record-and-proceed` / `halt`), so the match is literal, not judgment-call
vocabulary translation:

- **I'll just do** (`decide-alone`) — just decide, keep going, no record.
- **I'll do and note** (`record-and-proceed`) — append a record to
  `work/<task-id>/deviations.md` (use
  `${CLAUDE_SKILL_DIR}/../../templates/deviations.md`'s shape; tier
  `record-and-proceed`, status `resolved` immediately since proceeding *is*
  the resolution), then keep going.
- **I'll stop and ask before** (`halt`) — schema, public contracts, new
  dependencies, auth logic, or anything protected-path (the hooks enforce
  the file-level cases independently). Append a deviations.md record with
  status `open`, stop implementing, and tell the human what you need
  resolved. This is a legitimate non-recurring touchpoint (build prompt
  §2.7) — it does not happen on every task, only when reality diverges
  from the plan in a halt-tier way.

**Circuit breaker:** count every deviations.md record regardless of tier.
On the third for this task, the plan is invalidated — `git stash push -u -m
"spine: circuit breaker, work/<task-id>"` to preserve what you'd built
without losing it, write `state` back to `research`, bump
`work/<task-id>/ledger.json`'s `deviation_count` (see §6), `registry-sync
<task-id>`, and tell the human plainly: three wrong guesses means the
research was wrong once, not that each guess should be patched forward.
Fresh research is required before re-planning.

If a resolution (halt or otherwise) cites a `docs/charter.md` line, it must
end amend-or-reaffirm: the human either edits that charter line or
reaffirms it as-is, dated, and the deviations.md resolution records which.

## 5. Verify and ship

**Flag check first**, same rule as step 3. Implementation acceptance
checks (from the plan) should already pass before you move on — check them
yourself first; don't hand a known-broken diff to `/verify`. Then: `ledger
mark <task-id> verify`, write `state` = `verify`, `registry-sync
<task-id>`, invoke `/verify` (Skill tool) with the task ID.

If `/verify` reports the floor failed: fix it (back to implementation,
same task, not a new deviation by itself unless the fix itself diverges
from the plan) and re-run `/verify`. If it passed: write `state` = `ship`,
`registry-sync <task-id>`, invoke `/ship` (Skill tool) with the task ID.
`/ship` handles the merge gate, the commit trailer, the briefing, and
clearing `.spine/current-task` (Extension C additions to `/ship` itself —
ship-time re-grounding, second-approver, its own final registry sync —
land in Phase C, not here).

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
