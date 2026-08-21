---
name: task
description: Run the spine on a piece of work — classify, research, plan, approve, implement, verify, ship. The default way any non-trivial change gets made in a spine-installed project.
disable-model-invocation: true
argument-hint: [description of the work] [--milestone <milestone-id>]
---

You are running `/task`, the spine (build prompt §2.2). `$ARGUMENTS` is the
task description as given, plus an optional `--milestone <milestone-id>`.

**If the description is empty, try auto-continue before asking for one.**
First, the normal resume check still wins: if `.spine/current-task` already
exists, this is not an empty-description case at all — skip straight to
**Resuming** below and ignore everything in this bullet. Only when there is
no active task and no description was typed, run:

```
${CLAUDE_SKILL_DIR}/../../scripts/next-milestone-task --project <project root> \
  [--milestone <id> if one was given]
```

This is a single deterministic call instead of hand-scanning every
`work/M*/milestone.md` and every member task's own `state` file via
separate reads — same completeness test `/roadmap` and `/ship` §3b use
(every `## Member tasks` entry a real task-id, every one of those tasks'
own `state` reading `done`), computed once in the one place, live, so it
can't drift the way a cached "last completed milestone" pointer could
once a `## Closes milestone gap` splice (§3 below) reopens a milestone
that already looked complete. Its one-line output branches four ways:

- **`TBD <milestone-id> <description>`** — propose it and stop: "Continue
  with `<milestone-id>`'s next task: `<description>`?" — before doing
  anything else, Step 0 included. This is a real touchpoint, not a
  courtesy notice: nothing has been created yet (no task folder, no
  git-identity resolution, no ledger), so it's the cheapest possible point
  to catch a wrong guess, one keystroke against retyping the whole
  description by hand. On confirmation, proceed exactly as if the human
  had typed `<that description> --milestone <that-milestone-id>`. On
  rejection or a correction, use what the human says instead (a different
  entry, a different milestone, or a hand-typed description) — don't
  re-guess.
- **`BLOCKED <milestone-id> <blocking-token>`** — that milestone's next
  queued task can't start yet: its immediate predecessor (`<blocking-token>`
  is a task-id, or the literal `TBD` if that predecessor hasn't even been
  created) isn't done. Say so plainly ("`<milestone-id>`'s prior task isn't
  done yet") and fall through to asking for a description — never skip
  ahead to a different milestone on your own.
- **`WAITING <milestone-id>`** — that milestone is mid-flight (every member
  task already has a real id, none are `TBD`) but nothing in it is done
  yet either, so there's nothing queued to propose. Say so plainly and
  fall through to asking for a description.
- **`NONE`** — no milestone exists yet, or every one found is already
  complete. Fall through to asking for a description, saying briefly why
  auto-continue didn't fire.
- If the script could not run at all, this is the "could not run" case the
  tooling-gap discipline (header note below) distinguishes from a real
  result — but no task folder exists yet at this point for that discipline's
  usual `notes.md` line, so just say so plainly to the human ("couldn't
  check for a queued milestone task") and fall through to asking for a
  description; don't treat a script that couldn't execute as "nothing
  queued."

**Step 0 — core version check (Extension C §2.1, the "cheap session-start
check").** No SessionStart-shaped hook exists to carry this — the
manifest forbids a new one — so it lives here, the one recurring entry
point every real task passes through:

```
$(readlink -f "${CLAUDE_SKILL_DIR}")/../../scripts/setup --check --project <project root>
```

(`readlink -f` resolves the symlink so this works whether the skill is
loaded from a direct checkout or a `.claude/skills/` symlink — `realpath`
is an acceptable fallback if `readlink -f` is unavailable.)

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

**If `--milestone <id>` is given:** locate `work/<id>/milestone.md` as
follows — **if `workspace.json` exists at the project root**, probe in
order: (1) `work/<id>/milestone.md` at the workspace root; (2)
`<member-repo-path>/work/<id>/milestone.md` for each repo in
`workspace.json`'s `repos` array, in listed order; use the first path that
exists. If none exists, the milestone is new — create it at the workspace
root. **If `workspace.json` is absent**, read/create `work/<id>/milestone.md`
at the project root as always. All reads and bookkeeping writes below
(TBD-replacement, gap edits) happen at the resolved path, never silently
re-rooted to the workspace root.

Read the resolved `milestone.md` (design-stage extension,
`core/templates/milestone.md`) before classifying — its `## Member tasks`
list, `## Inter-task contracts` (what a prior member task in this milestone
left true, which this task may assume without re-verifying), and `##
Capability targets` all become planning context for every phase below. Once
this task's ID is generated (step 1), replace this milestone's first
still-`TBD` member-task line with the real task ID (`Edit` on the resolved
`milestone.md` path — this is bookkeeping, not a phase artifact). **Do
this before writing `work/<task-id>/state` in step 1, not after** —
`phase-gate` only restricts `Edit`/`Write` to a task's own
`work/<task-id>/` once that task has a `state` file reading `research` or
`plan`; with no `state` file written yet, this edit is simply outside the
hook's gating window, not something the hook has to carve out a special
case for. Record `work/<task-id>/milestone` = `<id>`, one line, so `/ship`
(final member task's own done-definition check) and a resumed session both
know this task belongs to a milestone without re-parsing `$ARGUMENTS`.

State lives in four places, and every phase transition below updates them
— they are not decoration, the hooks (`core/hooks/phase-gate`,
`path-escalate`) read them on every `Edit`/`Write`:

- `.spine/current-task` — the active task ID, one line. Absent = no active
  task = Class 0 default.
- `work/<task-id>/state` — the current phase name, one line.
- `work/<task-id>/class` — `0`, `1`, or `2`, one line.
- `work/<task-id>/autonomy` — `guided`, `checkpointed`, or `auto`, one line
  (absent = `guided`, the default). This is the Phase 4 dial for *how many
  human stops* the flow has, orthogonal to `class` (which sets *how much
  verification*). **Blast radius caps autonomy, never the reverse:** a Class 2
  task is always `guided`; a Class 1 task may be `auto`/`checkpointed`/`guided`;
  Class 0 is `traced` (no task folder — §1). `/intake` sets this and enforces
  the ceiling; §3's escalation re-enforces it. `auto` runs with no scheduled
  stops — only the same tripwires every task has (a `halt`-tier deviation, a
  mid-stream escalation, the circuit breaker, a verify `FAIL`) pull the human
  in. Checks (floor, adversaries) never scale down with autonomy; only stops do.

**The registry (Extension C §2.2/§2.3), sibling files alongside the three
above — never crammed into `state` itself**, which every hook and skill
above already reads as a bare one-line phase name:

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
`${CLAUDE_SKILL_DIR}` is a placeholder you expand to this skill's own
directory; hand the resulting path — including the `../../` — to the shell
verbatim. Do **not** lexically collapse `skills/task/../..` to `.claude/`:
`.claude/skills/task` is a symlink into the spine core checkout, so the shell
must resolve `../../` against the symlink's real target (`<spine>/core/...`).
Collapsing it as text yields a nonexistent `.claude/scripts/...` path and a
"no such file" error.

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

**If `.spine/current-intake` exists, this task came through `/intake`**
(`core/skills/intake/SKILL.md`). Read it — `{ticket, class, autonomy, description,
class_below_recommended, recommended_class}` — and treat the class as already
confirmed: the human confirmed it in `/intake`'s menu, so do **not** re-suggest
or re-prompt the class below. Use `description` as the task description if
`$ARGUMENTS` carried none. Proceed straight to task-ID generation with that
class. Two things ride along in the setup step below: write
`work/<task-id>/ticket` = the ticket key (one line; omit the file if the ticket
was null), which `/ship` reads for the `Spine-Ticket:` trailer; write
`work/<task-id>/autonomy` = the handoff's `autonomy` (a Class 2 task is forced
to `guided` regardless of what the handoff says — the ceiling); for a direct
`/task` with no handoff, write the autonomy the human just chose in the step
above (Class 2 / a downgraded task ⇒ `guided`); and if
`class_below_recommended` is true, when you record `class_declared` (§6) also
`ledger set <task-id> class_downgraded_from <recommended_class>` and add a
`notes.md` line — the downgrade stays visible without consuming a
circuit-breaker slot (it is a classification choice, not a plan-vs-reality
deviation, so it is **not** a `deviations.md` record). Then **delete
`.spine/current-intake`** and continue to §2. The class-suggestion list below is
only for a `/task` invoked directly, with no intake handoff.

Every task gets a class, and the human confirms it — not the model alone
(build prompt §2.7). Suggest one, don't decide it unilaterally (for a direct `/task` you also
confirm an autonomy right after the class — see "Autonomy for a direct
`/task`" below; `/intake` instead proposes it from a code-grounded pre-scan):

- **Class 0 (trivial):** suggest when the change looks like it will touch
  ≤2 files, ≈15 lines or fewer (or this project's `.spine/profile.json`
  `class0_max_files`/`class0_max_lines` if set), introduces no new public
  symbol, and (check
  against `.spine/protected-paths.conf`) touches no protected path. No work
  folder, no per-task `ledger.json`, no phases — but not invisible: make the
  edit, then leave a **trace** (traced-trivial). Derive the ticket from the branch —
  `${CLAUDE_SKILL_DIR}/../../scripts/ledger ticket-from-branch --project <project
  root>` — and if it returns a key, commit the edit carrying a `Spine-Ticket:
  <key>` trailer (composes with any subject convention, `core/ADAPTER-CONTRACT.md`
  §6) and record it: `${CLAUDE_SKILL_DIR}/../../scripts/ledger trace <key> "<one
  line: what changed>" --project <project root>`. That's the whole ceremony — one
  commit, one trace line, no folder, no docs. If no ticket is derivable (genuinely
  off-ticket), make the edit and skip the trailer/trace; spine doesn't chase
  off-ticket one-offs — they stay visible via the org's own commit convention.
  The backstop is
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

**Autonomy for a direct `/task`** (no intake handoff; the choice only exists at
Class 1 — Class 0 is `traced`, Class 2 is always `guided`, the ceiling). Once the
class is confirmed, ask the human how autonomous the flow should run: `guided`
(stop at each phase — the default and the safe choice), `checkpointed` (approve
the plan, then one finish action), or `auto` (no scheduled stops; you review the
finished PR). Default to `guided` if they express no preference. **If you had
suggested a higher class than the human chose** (e.g. you suggested Class 2, they
picked Class 1), lean `guided` and say why — a change you read as
higher-blast-radius is exactly the kind to keep a human in the loop on, even at
the class they chose. Cap the offer at `.spine/profile.json`'s `autonomy_ceiling`
if set. You'll write the result to `work/<task-id>/autonomy` in the setup step
below, the same place the class file is written. (`/intake` proposes this from
its pre-scan; a direct `/task` does none, so it simply asks.)

Once confirmed, for Class 1/2: generate the task ID
`<YYYYMMDD>-<kebab-slug>` (today's date, a short slug from the description),
`mkdir -p work/<task-id>`, write `.spine/current-task`, write
`work/<task-id>/class`. **If `--milestone <id>` was given, do this next
step now, before writing `work/<task-id>/state`** — write
`work/<task-id>/milestone` = `<id>` and replace this milestone's first
still-`TBD` member-task entry with `<task-id>` in `work/<id>/milestone.md`,
per this skill's own header note above (`phase-gate` only gates
`Edit`/`Write` once a `state` file exists and reads `research`/`plan`; no
`state` file yet means this edit is a genuine no-op for the hook to
evaluate, not an exemption it has to special-case). Only after that: write
`work/<task-id>/state` = `research`, then
`${CLAUDE_SKILL_DIR}/../../scripts/ledger init <task-id>` and
`ledger mark <task-id> classify`. If `ledger init` could not run at all,
this is the ledger-itself-unreachable case in the tooling-gap discipline
above — hand-author the `ledger.json` stub (`hand_tracked: true`) right
now, at task creation, rather than waiting for a later phase to notice; a
task that never gets a ledger.json at all is invisible to `/costs`, not
just degraded.

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
silently contradicted. Also check that milestone's `## Known gaps for
future member tasks`: if this plan's approach actually closes one of its
`gap-<n>` entries, add the `## Resolves known gaps` section per
`core/templates/plan.md` naming it — this is what lets `/ship` §3c remove
the entry once the task ships; a gap this plan resolves without
citing it stays listed, which is a missed cleanup, not a wrong one, so
don't invent a citation just to clear the section. If any decision from
`/design` grounds this plan, add the `## Grounds on decisions` section per
`core/templates/plan.md`.

**If this task does *not* already belong to a milestone** (`work/<task-id>/
milestone` unset — no `--milestone` was given), still check whether this
plan's own scope is required to make some existing milestone's `##
Done-definition` true despite not being one of that milestone's listed `##
Member tasks` — the shape a prior member task's own `briefing.md` flagging
a real gap in its follow-ups most often takes. If so, write the `##
Closes milestone gap` section per `core/templates/plan.md` naming that
milestone, then act on it right now, before presenting the plan: resolve
`work/<id>/milestone.md`, append a new numbered entry to its `## Member
tasks` with this task's own real id (no `TBD` — the task already exists)
and a one-line description drawn from `## The gist`'s first sentence, and
write `work/<task-id>/milestone` = `<id>`. Say this plainly when presenting
the plan for approval — "this also closes M<n>'s done-definition gap,
splicing it in as member task <k>" — same visibility standard as any other
milestone-affecting edit this skill makes. This is the fix for the exact
blind spot a real ad-hoc gap-closing task exposed: without it, a task that
genuinely closes a milestone's done-definition gap never gets a
`work/<task-id>/milestone` pointer, so none of `/ship`'s §3a/§3b/§3c
milestone bookkeeping ever engages for it and the milestone's own record
never shows a 5th task was actually required to reach "done." If the named
milestone doesn't resolve to a real `milestone.md`, or `work/<task-id>/
milestone` was already set to a *different* id than this section names,
that's a conflict — surface it to the human, never silently pick one or
fabricate the file.

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
workspace root for a multi-repo task — one class for the whole task), **and
rewrite `work/<task-id>/autonomy` = `guided`** (the ceiling — a Class 2 task
is never `auto`/`checkpointed`; escalation pulls the human back in), and
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

**If `work/<task-id>/autonomy` is `auto`, there is no plan-approval stop.**
Write the plan exactly as above — it is still written, and `/ship` attaches it
to the PR for review, trading pre-implementation plan review for PR-time review
(the disclosed `auto` tradeoff — see `docs/tradeoffs.md`). Record `approval.json` as a self-approval with `"autonomy": "auto"` set,
`registry-sync`, and proceed straight to §4. This can only happen at Class 1
(the ceiling); if §3's protected-path check just auto-escalated this task to
Class 2, `work/<task-id>/autonomy` was set to `guided` above, so this branch no
longer applies and you fall through to the stop below. For `checkpointed` and
`guided`:

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
<task-id>`.

**This phase's shape depends on `work/<task-id>/autonomy`** (absent =
`guided`). The independence that `/verify` protects comes from the falsifier
and security agents being *fresh, isolated subagents* — never from who typed
the command — so `checkpointed`/`auto` preserve it while removing the human
relay the engineer explicitly delegated by choosing that autonomy at `/intake`.

**guided** — `/verify` and `/ship` both carry `disable-model-invocation: true`,
so this skill cannot call either via the Skill tool; only the human literally
typing `/verify <task-id>` (or `/ship <task-id>`) gets through. Tell the human
plainly: implementation is ready, please run `/verify <task-id>` yourself. Then
stop and wait — this session does not proceed to ship on an unverified diff, the
same waiting posture step 3 uses for a Class-2 second approver. When resumed,
**read `work/<task-id>/verify.md` directly** (its `Result:` line reads `PASS` or
`FAIL` verbatim). If `FAIL`: fix it (back to implementation, same task) and ask
the human to re-run `/verify`. If `PASS`: write `state` = `ship`, `registry-sync`,
then ask the human to run `/ship <task-id>` and wait the same way.

**checkpointed** — one human action closes out the task instead of two. Present a
single **finish** confirmation ("implementation's ready and the plan's acceptance
checks pass — verify and ship?"). On the human's go, **follow
`core/skills/verify/SKILL.md`'s steps inline** — read that file and execute its
steps in this session. This is *not* a Skill-tool invocation, so
`disable-model-invocation` does not block it (that flag blocks the tool call, not
following the written procedure); the human authorized it with the finish
confirmation. Read the resulting `verify.md` `Result:`. On `PASS`, **follow
`core/skills/ship/SKILL.md`'s steps inline** the same way. On `FAIL`, fix and
re-run verify inline — no new human stop unless a `halt`-tier deviation opens
(§4).

**auto** — no scheduled human stop. **Follow `core/skills/verify/SKILL.md` inline**
(same mechanism as `checkpointed`; the falsifier's stub-out probe is *mandatory*
in this mode — it is the partial backstop for the plan review `auto` skipped).
Read `verify.md`'s
`Result:`. On `PASS`, **follow `core/skills/ship/SKILL.md` inline**, which for an
`auto` task opens a **draft PR** (never merges — `core/skills/ship/SKILL.md` §5a)
and stops at "PR ready for review." On `FAIL`, this is an
*exception* stop: go back to implementation, fix, re-run verify inline; if the fix
hits a `halt`-tier decision or trips the circuit breaker (§4), stop and pull the
human in exactly as §4 says. The human's single touchpoint is reviewing the
finished PR.

For every mode, `/ship` (however it runs) handles the merge gate, the commit
trailer(s), the briefing, and clearing `.spine/current-task`.

When resumed after `/ship`, confirm it actually completed by checking that
`.spine/current-task` no longer names this task (`/ship` clears it on
success) and that `work/<task-id>/briefing.md` exists, rather than taking
the human's word alone.

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
