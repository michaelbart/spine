---
name: task
description: Run the spine on a piece of work — classify, research, plan, approve, implement, verify, ship. The default way any non-trivial change gets made in a spine-installed project.
disable-model-invocation: true
argument-hint: [description of the work] [--milestone <milestone-id>]
---

You are running `/task`, the spine. `$ARGUMENTS` is the
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
  git-identity resolution), so it's the cheapest possible point
  to catch a wrong guess, one keystroke against retyping the whole
  description by hand. On confirmation, proceed exactly as if the human
  had typed `<that description> --milestone <that-milestone-id>`. On
  rejection or a correction, use what the human says instead (a different
  entry, a different milestone, or a hand-typed description) — don't
  re-guess. Ask it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

  <!-- touchpoint:start confirm -->
  > **Start the next planned task in `<milestone title>`: "<description>"?** It's next in the plan's order, and nothing is created until you say yes.
  > **Yes** (recommended) — I begin researching it. **No** — tell me a different task or milestone instead.
  <!-- touchpoint:end -->
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
- If the script could not run at all, this is a tooling gap — say so
  plainly to the human ("couldn't check for a queued milestone task") and
  fall through to asking for a description; don't treat a script that
  couldn't execute as "nothing queued."

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
maintainer has bumped the pin (`core/skills/update/SKILL.md` has the full
upgrade workflow). If `setup` itself could not run at all,
this is the tooling-gap discipline below's "could not run" case —
note it and proceed, don't treat an unreachable check as a passing one.

**Multi-repo (Extension B)**: if `workspace.json` exists at the project
root, this session's own project root *is* the workspace root, and this
one `/task` invocation is the single task folder, single plan, single
human approval for however many member repos the change touches — never
a separate `/task` per repo. Every step below runs
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
exists. **If none exists, the milestone is new — before creating it, stop
and confirm with the human rather than silently creating an unplanned
milestone.** This is the point `/roadmap`'s own sequencing and gap-absorption
(`core/skills/roadmap/SKILL.md` §1a/§3) would normally already have run —
skipping straight to task creation is exactly the path that lets Known Gaps
entries never get absorbed anywhere. Ask it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start confirm -->
> **Create `<milestone title>` here, or plan it first with `/roadmap`?** It has no plan file yet, and creating it here skips the step that sorts out leftover issues.
> **Plan it first** (recommended) — you run `/roadmap`, then come back. **Create it here** — I make a minimal milestone and ask about leftover issues from finished ones.
<!-- touchpoint:end -->

This is informational, not a
hard block — same "never a gate" stance `/ship` takes on the identical
suggestion — the human may legitimately want an ad hoc milestone id. If the
human says proceed: **also check every already-shipped milestone**, not
just the immediately preceding one — read each `work/M*/milestone.md`
directly, checking for open entries in its `## Known gaps` section. For
every shipped milestone with open entries, read them aloud and ask what to
do with each: carry it verbatim into `<id>`'s own Known Gaps section (same
id, `source`, prose — the append-verbatim convention
`core/skills/roadmap/SKILL.md` §3 uses), fold it into this task's own scope
instead, explicitly decline it (say so, move on — never silently drop, same
completeness standard `/roadmap` §4 holds itself to), or defer it — leave
it exactly where it is and say nothing more now. Deferring is a safe,
legitimate answer here (unlike declining, it isn't final): the gap stays in
its original milestone's own Known Gaps section and this same stop will
re-ask about it at the next new-milestone creation.

Ask once per leftover issue, with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start -->
> **Deciding:** what to do with one problem an earlier milestone knowingly left open. It's yours because nobody fixed it, and only you can say whether it belongs to this work.
> **Need to know:** "<the issue in one plain sentence>", left by `<earlier task>`. Nothing is lost; it stays listed in its old milestone until you decide.
> **Recommend:** <carry it | fold it in | ask me later> — <one-line reason>
> 1. **Carry it forward** — next: I copy it unchanged into this milestone's list of known issues; cost: none; undo: yes
> 2. **Fold it into this task** — next: this task's scope grows to fix it; cost: extra work in this task; undo: yes, until the plan is approved
> 3. **Decline it** — next: I record that you chose not to fix it; cost: it stays unfixed for good; undo: no, this one is final
> 4. **Ask me later** — next: it stays where it is and I ask again when the next milestone starts; cost: none; undo: yes
> **Safe to ignore:** the other leftover issues; each is asked separately.
<!-- touchpoint:end -->

Only then create
`work/<id>/milestone.md` at the workspace root, seeded from
`core/templates/milestone.md` plus whatever gap entries were carried
forward (bump `next-gap-id` past the highest carried id). **If
`work/<id>/milestone.md` already exists, none of this applies** — either
`/roadmap` already ran and did it, or an earlier task in this same milestone
already did. **If `workspace.json` is absent**, read/create
`work/<id>/milestone.md` at the project root as always (the new-milestone
stop above applies here too). All reads and bookkeeping writes below
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
them, don't paraphrase), in this form (per `core/templates/human-touchpoint.md`), and stop:

<!-- touchpoint:start short -->
> **What happened:** <who> changed <what> at <when>, and the plan relied on it (flag fields quoted).
> **What it means for you:** I paused before the next step because the plan may no longer match reality. Nothing has been changed.
> **To continue:** look at that change; if it doesn't matter, mark the flag acknowledged in `work/<task-id>/flags.json` and tell me to continue.
<!-- touchpoint:end --> The human resolves it the same way any
halt-tier deviation resolves — by acting on the information, then editing
that flag entry (`"acknowledged": true`, `"acknowledged_at"`,
`"acknowledged_by"` set) — never by silently clearing it or advancing
around it. This is the mechanism `core/scripts/propagate`'s flags exist to
be *for*; a flag nothing ever reads back would be exactly the "manufactures
confidence" failure shape a hook that doesn't fire produces — `/task` is
what performs every phase transition, so `/task` is what
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

**Resuming, or a second task in a worktree (Extension D — experimental):**
if `.spine/current-task` already exists,
first check whether this invocation actually names *new* work — a real
description was typed (`$ARGUMENTS` non-empty) or `.spine/current-intake`
exists — as opposed to a bare `/task` with nothing new to say, which always
means resume. **Bare `/task`, nothing new:** read the existing task's `state`
and `class` and offer to resume it where it left off rather than starting a
new one — do not silently abandon it. Re-derive nothing from memory; a fresh
session has none, so read `work/<task-id>/{research,plan,deviations}.md`
before proceeding. **A new description (or intake handoff) was given while a
task is already active here:** don't silently swallow it into a resume offer
of the *other* task — that discards what the human just typed. Instead, first
check whether this working directory is itself already a spine-created
worktree (cheapest signal: the cwd path contains `/.claude/worktrees/` —
`EnterWorktree`'s own convention); if so, skip straight to the bare-`/task`
resume behavior above regardless — a worktree task never offers to spin up a
worktree of its own, that's how nesting is avoided. Otherwise, ask plainly:
`"<existing-task-id>` is active here (phase `<state>`). Start `<new
description>` in a separate worktree instead of interrupting it, or
resume/switch to `<existing-task-id>` here instead?"` Three real answers,
never silently pick one:

- **Resume/switch to `<existing-task-id>`** — fall through to the bare-`/task`
  resume behavior above, ignoring the new description (the human chose the
  existing task instead).
- **New worktree** —
  ```
  EnterWorktree({ name: <a kebab-slug derived from the new description> })
  ```
  which switches this session's own working directory. Immediately, before
  anything else runs in the new location:
  ```
  ${CLAUDE_SKILL_DIR}/../../scripts/setup --project <the new worktree's path>
  ```
  (`${CLAUDE_SKILL_DIR}` still resolves correctly here — it names this skill
  file's own directory, unaffected by the session's cwd changing; pass the
  new path explicitly via `--project` rather than relying on cwd.) `setup` is
  idempotent and already handles being re-run — this call is what regenerates
  the worktree's own `.claude/skills|agents|rules|hooks` symlinks and
  `.claude/settings.local.json`, none of which `git worktree add` brings along
  on its own since they're gitignored/machine-local in the original checkout.
  Treat a `setup` failure here as a genuine stop, not a tooling-gap-and-continue
  — running any further spine logic in a worktree with broken hook wiring
  would silently defeat `phase-gate`/`path-escalate`/`dep-gate` for this task's
  entire lifetime, not just degrade one check. Once `setup` succeeds, restart
  this skill's own procedure from the top, from the new working directory, with
  the same description — a `.spine/current-task` doesn't exist yet there, so
  this time nothing routes back into this Extension D branch.
- **Something else the human types** — follow it; don't re-guess.

At `/ship`'s close-out, a task that ran in a spine-created worktree offers to
clean it up — see `core/skills/ship/SKILL.md` §6. Known limitation, disclosed
in `docs/tradeoffs.md`: `EnterWorktree` puts each worktree on its own new
branch, so this task's `registry-sync` pushes to that branch, not the shared
default branch — a colleague's `claims-check` won't see this task's
`work/<task-id>/` folder until that branch merges. Harmless for the same
engineer's own next `/task` (both worktrees share one local `.git`); it only
matters for cross-engineer visibility, which this experiment isn't targeting.

Scripts referenced below live at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`.
Templates live at `${CLAUDE_SKILL_DIR}/../../templates/<name>`.
`${CLAUDE_SKILL_DIR}` is a placeholder you expand to this skill's own
directory; hand the resulting path — including the `../../` — to the shell
verbatim. Do **not** lexically collapse `skills/task/../..` to `.claude/`:
`.claude/skills/task` is a symlink into the spine core checkout, so the shell
must resolve `../../` against the symlink's real target (`<spine>/core/...`).
Collapsing it as text yields a nonexistent `.claude/scripts/...` path and a
"no such file" error.

**Tooling-gap discipline (applies to every script invocation below):**
every time you invoke a core script, distinguish three outcomes — "ran and
passed," "ran and failed" (a real result, act on it normally), and **could
not run at all** (blocked, denied, or errored before the script's own logic
executed). On "could not run": append a line to `work/<task-id>/notes.md`
(create it, header `# Notes`, if it doesn't exist yet):
`TOOLING GAP: <script> could not run — <one-line consequence>.` Be concrete
about the consequence. Never let a could-not-run script silently read as
"nothing to report." This carries forward into `verify.md`'s own "Tooling
gaps" section.


**Voice.** Every message this skill leaves for the human follows
`core/templates/human-touchpoint.md`. Before sending one, run its "Before you
send" list: gloss or drop internal names, IDs and commit hashes, keep one
decision per message, and put surprises first.

## 1. Classify — the first recurring human touchpoint

**If `.spine/current-intake` exists, this task came through `/intake`**
(`core/skills/intake/SKILL.md`). Read it — `{ticket, class, autonomy, type,
description, class_below_recommended, recommended_class}` — and treat the
class as already confirmed: the human confirmed it in `/intake`'s menu, so do
**not** re-suggest or re-prompt the class below. Use `description` as the
task description if `$ARGUMENTS` carried none. Proceed straight to task-ID
generation with that class. Two things ride along in the setup step below:
write `work/<task-id>/ticket` = the ticket key (one line; omit the file if
the ticket was null), which `/ship` reads for the `Spine-Ticket:` trailer;
write `work/<task-id>/autonomy` = the handoff's `autonomy` (a Class 2 task is
forced to `guided` regardless of what the handoff says — the ceiling);

**Ticket branch (Extension F, experimental — `/intake`-originated, Class 1/2
only), done here rather than in `/intake` itself:** this point comes after the
Resuming/Extension D check above already resolved where this task actually
runs (either this directory had no other active task, or the human just
arrived in a fresh worktree) — the one place a branch switch can't collide
with another task's uncommitted work sitting in the same tree. If `ticket` is
non-null, first check whether the current branch already resolves to this
same key: check whether `git rev-parse --abbrev-ref HEAD` already contains
the ticket key as a substring. A match means the human already branched by
hand — nothing to do. Otherwise, before generating the task ID below (so the task
folder's own commits land on the right branch from the start):

**Compute `<branch>` from this project's own naming convention, not a fixed
shape.** Read `.spine/branch-naming.conf` if it exists (one line, a template
using `{ticket}`, `{slug}`, `{type}`, `{user}`); if absent, the template is
`{ticket}-{slug}` (this extension's original, unconfigurable default —
unchanged for any project that never opted in). Substitute: `{ticket}` = the
ticket key; `{slug}` = a kebab-slug of the description; `{type}` = the
handoff's `type` (`feature`|`fix`) — a template containing `{type}` with no
`type` in the handoff (a stale pre-upgrade `.spine/current-intake`, or this
call site reached with no handoff at all) is a tooling gap: fall back to
`feature` and don't halt the branch creation over it, but remember to add a
`TOOLING GAP:` line to `notes.md` once the task folder exists a few steps
below — the folder doesn't exist yet at this point in the flow, this can't
be logged immediately; `{user}` = `git config user.name`, slugified the
same way as `{slug}`.
**A committed `.spine/branch-naming.conf` whose template doesn't contain
`{ticket}` is malformed** — the substring check above depends on the ticket
key actually appearing in the branch name — fail loud (tooling gap, ask the
human) rather than silently using it or silently falling back to the
default; this should have been caught at write time (`core/skills/bootstrap/
SKILL.md` §4 / `core/skills/adopt/SKILL.md` §3) and reaching it here at all
means that validation was skipped or the file was hand-edited since.

- A local branch already named `<branch>` exists (`git rev-parse --verify
  --quiet refs/heads/<branch>`) — check it out, don't recreate it (a resumed
  ticket, or a colleague's branch pulled locally).
- Otherwise a remote-tracking one exists (`git rev-parse --verify --quiet
  refs/remotes/<remote>/<branch>`, `<remote>` read the same way as below) —
  `git checkout -b <branch> <remote>/<branch>`.
- Otherwise, create it fresh from current HEAD: `git checkout -b <branch>`.
  Then, if the branch you were just on has a configured remote
  (`git config branch.<previous-branch>.remote`), immediately `git push -u
  <remote> <branch>` — this is what makes `registry-sync`'s later pushes work
  at all (it resolves the push target from `branch.<branch>.remote`, which
  only exists once something sets it) and what gives `open-pr`'s "uses the
  current branch when `SPINE_PR_HEAD` is unset" (`core/ADAPTER-CONTRACT.md`
  §3.5) a real head branch to open a PR from, instead of silently assuming
  the human already branched by hand. No remote configured on the previous
  branch: skip the push, stay local — same "solo/single-machine project"
  no-op `registry-sync` itself already treats as normal, not a gap. A push
  failure here (network, permissions) is non-fatal — note it and continue on
  the local branch; the next `registry-sync`/`/ship` push retries it
  naturally once the human resolves it by hand.

One more thing rides along in the setup step below: for a direct
`/task` with no handoff, write the autonomy the human just chose in the step
above (Class 2 / a downgraded task ⇒ `guided`); and if
`class_below_recommended` is true, add a `notes.md` line recording the
downgrade — the downgrade stays visible without consuming a circuit-breaker
slot (it is a classification choice, not a plan-vs-reality deviation, so it
is **not** a `deviations.md` record). Then **delete `.spine/current-intake`**
and continue to §2. The class-suggestion list below is
only for a `/task` invoked directly, with no intake handoff.

Every task gets a class, and the human confirms it — not the model alone.
Suggest one, don't decide it unilaterally (for a direct `/task` you also
confirm an autonomy right after the class — see "Autonomy for a direct
`/task`" below; `/intake` instead proposes it from a code-grounded pre-scan):

- **Class 0 (trivial):** suggest when the change looks like it will touch
  ≤2 files, ≈15 lines or fewer (or this project's `.spine/profile.json`
  `class0_max_files`/`class0_max_lines` if set), introduces no new public
  symbol, and (check against `.spine/protected-paths.conf`) touches no
  protected path. No work folder, no phases — but not invisible: make the
  edit, then commit it carrying a `Spine-Ticket: <key>` trailer if a ticket
  key is derivable from the branch name (split on `-`, match against
  `.spine/ticket-pattern.conf` if present). If no ticket is derivable
  (genuinely off-ticket), make the edit and skip the trailer; spine doesn't
  chase off-ticket one-offs — they stay visible via the org's own commit
  convention. The backstop is `path-escalate`: with no active task it
  defaults to class 0, so if the edit turns out to touch a protected path,
  the hook halts it and you tell the human plainly: "this stopped being
  trivial" — then restart as a real task.
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
ceremony" / "a standard change" / "a high-risk change: a second person
approves the plan"), not by number, unless glossed:

<!-- touchpoint:start confirm -->
> **Run this as a <trivial | standard | high-risk> change?** <One sentence: what it touches, and what happens automatically if that reading turns out wrong.>
> **Yes** (recommended) — <what happens>. **No, <the alternative in plain words>** — <what that costs you>.
<!-- touchpoint:end -->

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
its pre-scan; a direct `/task` does none, so it simply asks.) Ask it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start -->
> **Deciding:** how often I should stop and check with you on this task. It's yours because it trades your attention against how closely you watch the work.
> **Need to know:** The three settings are `guided` (I stop after every phase and wait for you), `checkpointed` (I stop for plan approval, then once more at the finish) and `auto` (I run without scheduled stops; you review the finished PR).
> **Recommend:** The setting called guided — it is the safe default; pick less stopping only if the change is small and easy to review.
> 1. **Stop after every phase** — next: I pause after research, plan and each step for your go-ahead; cost: the most of your attention; undo: yes, you can switch later
> 2. **Approve the plan, then one finish check** — next: I run to the end after you approve the plan, then ask once; cost: two stops; undo: yes
> 3. **No scheduled stops** — next: I run everything and you review the finished PR; cost: you find problems late; undo: yes, nothing merges without you
> **Safe to ignore:** the setting names; I'll use plain words.
<!-- touchpoint:end -->

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
evaluate, not an exemption it has to special-case). Only after that: write `work/<task-id>/state` = `research`.

**Registry init (Extension C §2.2), same step, before the first
`registry-sync`:** resolve owner identity —
`git config user.name` and `git config user.email`. **If either is empty,
stop before creating the task folder** and tell the human in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start short -->
> **What happened:** I can't find your git name and email, and I don't make one up.
> **What it means for you:** I haven't created anything yet; the task's records are signed with that identity.
> **To continue:** run `git config --global user.name '<you>'` and `git config --global user.email '<you@example.com>'`, then tell me to go on.
<!-- touchpoint:end --> (A fresh machine genuinely has neither set;
never fall back to `$USER`, hostname, or any other guess.) Otherwise write
`work/<task-id>/owner` = `<name> <email>`, one line. Write
`work/<task-id>/claims.json` from `core/templates/claims.json` with
`task_id` filled in and every array empty (populated for real at plan
approval, §3). Write `work/<task-id>/flags.json` = `[]`. Then:

```
${CLAUDE_SKILL_DIR}/../../scripts/registry-sync <task-id> --project <project root> --message "task: open <task-id>"
```

This is the literal "committed and pushed to it at task creation" the
registry requires — a task invisible to a colleague's `claims-check`
until `/ship` would defeat the entire mechanism, so this happens now, not
deferred to the end of the phase.

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

**UI-touching tasks: the handoff is required grounding.** If the task's
description or likely touch set includes a declared UI path or content
path (`.spine/ui-paths.conf`, `.spine/ui-content-paths.conf`), tell the
researcher to read and cite, in the header's `files:` list, the affected
screen's `docs/ui/screens/<id>.json` **and** each of that screen's
screenshots (every path in its `screenshots` map), plus
`docs/ui/components.md` where a component's behavior matters. A UI task
researched only from code is how content gets invented; the headers are
also what makes `check-stale` notice when a mockup is replaced.

The researcher's entire reply is the complete `research.md` content
(including its header) — write it verbatim to `work/<task-id>/research.md`.

## 3. Plan

**Flag check first** (per this skill's own header note on flag-blocked
advance): read `work/<task-id>/flags.json`; any unacknowledged entry halts
here, before anything else in this step. Write `state` = `plan`,
`registry-sync <task-id>`. First run
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

**False-positive rule (own bookkeeping is not drift).** If the only drifted
items are `work/<id>/milestone.md` for this task's own milestone, and every
changed line in `git diff <sha> -- work/<id>/milestone.md` is this task's own
classify-time replacement of the milestone's first `TBD` line with its task
id (the same clause-(d) reading `core/skills/ship/SKILL.md` §0 applies at ship
time), it is not drift. Keep the research: delete the `> **STALE**` banner
`check-stale` wrote into `research.md`, append one line to `notes.md`
("check-stale flagged only my own TBD-to-task-id edit in milestone.md;
treated as a false positive, research kept"), and go on to write the plan. Any
other drifted item, or any changed line you cannot attribute to that one edit,
means regenerate as above — never guess.

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
**200-line hard cap, comments included** — `wc -l < plan.md` it before
presenting; if it doesn't fit, the task splits into two, it does not get
compressed into unreadability. Multi-repo, additionally: write `## Ship order` the
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
protected path is hit" loop the mechanical form of "class escalation
composes as max across repos": there is no separate max
computation to write, it falls out of checking every entry regardless of
which repo it belongs to. If any match and `work/<task-id>/class` is not
already `2`, auto-escalate: rewrite the class file to `2` (one file, at the
workspace root for a multi-repo task — one class for the whole task), **and
rewrite `work/<task-id>/autonomy` = `guided`** (the ceiling — a Class 2 task
is never `auto`/`checkpointed`; escalation pulls the human back in), and
say so plainly when you present the plan — this is plan-triggered
escalation; it does not need a separate confirmation prompt beyond the
plan approval you're about to ask for anyway.

**Populate `work/<task-id>/claims.json` for real** (Extension C §2.2/§2.3),
now that a plan exists: `predicted_touch` from `## Predicted touch`
verbatim, `grounding_files`/`grounding_decisions` from `research.md`'s own
header, `contracts` from `## Contract change`'s named contract if present,
`updated_at` set to the current timestamp. `registry-sync <task-id>` — this is the version of claims.json a
colleague's `claims-check` (Phase C) sees; a claims.json still at its
empty classify-time skeleton would make every intersection check
vacuously pass, silently defeating the whole mechanism.

**UI-touching plans: run `content-sources-check` before presenting the plan.**

```
${CLAUDE_SKILL_DIR}/../../scripts/content-sources-check <task-id> --project <project root>
```

Exit 0 (including "not applicable") proceeds. Exit 1 means the plan's
`## Content sources` is missing, cites a nonexistent or untracked source,
omits a fixture/content file, or contains `source: none`. **Do not present
the plan.** For each `source: none`, stop and ask the human what the
content is or where it comes from (halt tier — the same stop-and-ask a
`halt` deviation is), then rewrite the entry as a real path or
`source: human — <what they said>` and re-run. This applies at every
class and autonomy, `auto` included: there is no self-approving your way
past content nobody defined. Ask it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start -->
> **Deciding:** what a piece of on-screen text should say, or where it comes from. It's yours because nothing in the project defines it, and I won't invent wording or sample data.
> **Need to know:** The plan puts "<the text or file>" on screen, and I found no source for it.
> **Recommend:** Just tell me the wording — it's the fastest way to be right.
> 1. **Give me the text** — next: I record it as your answer and carry on to the plan; cost: a minute; undo: yes
> 2. **Point me to where it lives** (a file or link) — next: I read it, cite it, and carry on; cost: a minute; undo: yes
> **Safe to ignore:** every entry that already has a source.
<!-- touchpoint:end -->

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
to override is itself the resolution) naming the conflicting task. If
clear (exit 0, warnings or not), proceed straight to presenting the plan.

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
iteration. Present it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start -->
> **Deciding:** whether I may start building this plan. It's yours because nothing changes until you approve, and after that I edit real files.
> **Need to know:** <the plan's bottom line in one plain sentence>. <If research staleness went unmeasured: "One caveat: I couldn't check whether my research notes are still current.">
> **Recommend:** <Approve | Revise> — <one-line reason from the plan's own risks>
> 1. **Approve** — next: I start building; cost: none; undo: yes, git can revert everything until it ships
> 2. **Ask for changes** — next: you tell me what to change and I show the plan again; cost: a few minutes; undo: n/a
> **Safe to ignore:** the file lists and internal labels in `plan.md`; the summary above is the whole decision.
<!-- touchpoint:end -->

**Record the approval** (Extension C §2.6) — `work/<task-id>/approval.json`:
`{"approver": "<git identity>", "at": "<iso8601>", "override": false,
"override_reason": null}`. `registry-sync <task-id>`.

- **Class 0/1**: the approver is whoever's session this is — resolve from
  `git config user.name`/`user.email` in *this* session, same as `owner`.
  Self-approval is expected and correct here; mandatory cross-review does
  not extend to Class 1 — it would recreate the review-bottleneck theater
  spine exists to escape.
- **Class 2**: the approver must be a *different* git identity than
  `work/<task-id>/owner`. This session cannot manufacture that identity —
  it can only ever resolve its own `git config`. So: if this session's own
  identity equals `owner`, **do not write `approver` as this session's own
  identity and call it approved.** Tell the human in this form (per `core/templates/human-touchpoint.md`):

  <!-- touchpoint:start short -->
  > **What happened:** this is a Class 2 (the highest-risk kind of change: needs a second person's approval) task and you own it, so you can't approve your own plan.
  > **What it means for you:** I've stopped before writing any code. Nothing is lost; I'll wait.
  > **To continue:** ask a colleague to pull this project, read `work/<task-id>/plan.md`, and record their approval from their own session (it uses their own git identity, so it can't be typed on their behalf); then tell me. Working solo? You can instead record a self-approval with a stated reason; it is flagged loudly in the briefing.
  <!-- touchpoint:end -->
  Then stop — this
  session waits (pull periodically, or the human says when it's done)
  rather than proceeding to implement on an unapproved Class 2 plan.
  **Override** (a genuine solo/vacation-coverage situation, same trust
  model as `/ship --bypass`): the owner may self-approve by writing
  `approval.json` with `"override": true` and a real
  `"override_reason"` — loud, not silent; `/ship` (Phase C's own
  extension) surfaces this in the briefing unconditionally, never treats it
  as an ordinary approval.

## 4. Implement

**Flag check first**, same rule as step 3. On approval: write
`state` = `implement`, `registry-sync <task-id>`. This is what unblocks
`phase-gate` — it only restricts writes during `research`/`plan`.

Work the plan's steps directly (you have full tool access again; `phase-gate`
no longer applies, `path-escalate`/`dep-gate` still do). For each decision
you hit, **first ask whether it's a setup event, not a deviation at all**:
did it teach you the plan's understanding of *the product* was wrong, or
only that this project's own tooling config (`.spine/adapters/*`,
`.spine/capabilities.json`, `.spine/protected-paths.conf`) was imperfect —
a latent adapter bug (e.g. pulling in a broken build target) with no
bearing on the plan's own reasoning, a stale capability status getting
corrected, and the like? The first is a real deviation, handled by the
three tiers below. The second is a **setup event**: fix it, append one
line to `work/<task-id>/notes.md` (create it, header `# Notes`, if it
doesn't exist yet) — `SETUP: <what was touched, what was wrong, how it was
fixed>` — and keep going. **Never a `deviations.md` record** — it doesn't
count toward the circuit breaker and doesn't appear in the briefing's
"What surprised us," because it isn't a plan-vs-reality mismatch about the
product; `core/skills/verify/SKILL.md` step 5 merges these `SETUP:` lines
into `verify.md`'s own "Setup events" section (mirroring exactly how a
`TOOLING GAP:` line already flows into that file's "Tooling gaps" section)
so it's still visible, never silent, just not conflated with a real
deviation. **A class escalation is never a setup event, even when it
traces to a plan-time check the escalation itself proves was a miss** — a
missed blast-radius call is exactly the "the plan's understanding of the
product was wrong" signal the circuit breaker exists to catch; it stays a
`halt`-tier deviation below, same as always.

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
- **I'll stop and ask before** (`halt`) — schema, public contracts, new
  dependencies, auth logic, or anything protected-path (the hooks enforce
  the file-level cases independently). Append a deviations.md record with
  status `open`, stop implementing, and ask the human in the form below. This is a legitimate non-recurring touchpoint — it does not
  happen on every task, only when reality diverges from the plan in a
  halt-tier way. Ask it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`). Cite no decision, gap or class
  id unless you say in a few words what it is:

  <!-- touchpoint:start -->
  > **Deciding:** <the specific thing that came up, e.g. "whether I may change the database layer">. It's yours because the plan said I'd check with you before touching <schema | public contracts | packages | login logic | protected files>.
  > **Need to know:** <what I found while building, in plain words>. <Why the plan didn't cover it>. <What each path would touch>. Nothing has been changed for this yet.
  > **Recommend:** <option> — <one-line reason>
  > 1. **<the smaller path>** — next: <what happens>; cost: <time or risk>; undo: <yes/no/how>
  > 2. **<the larger path>** — next: <what happens>; cost: <time or risk>; undo: <yes/no/how>
  > **Safe to ignore:** <records I'll update either way>
  <!-- touchpoint:end -->

**Circuit breaker:** count every deviations.md record regardless of tier.
On the third for this task, the plan is invalidated — `git stash push -u -m
"spine: circuit breaker, work/<task-id>"` to preserve what you'd built
without losing it, write `state` back to `research`, `registry-sync
<task-id>`, and tell the human plainly: three wrong guesses means the
research was wrong once, not that each guess should be patched forward.
Fresh research is required before re-planning.

If a resolution (halt or otherwise) cites a `docs/charter.md` line, it must
end amend-or-reaffirm: the human either edits that charter line or
reaffirms it as-is, dated, and the deviations.md resolution records which.

## 5. Verify and ship

**Flag check first**, same rule as step 3. Implementation acceptance
checks (from the plan) should already pass before you move on — check them
yourself first; don't hand a known-broken diff to `/verify`. Then: write
`state` = `verify`, `registry-sync <task-id>`.

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
then ask the human to run `/ship <task-id>` and wait the same way. Say
it in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start short -->
> **What happened:** the work is built and ready to be checked.
> **What it means for you:** I can't start the check myself; only you typing the command can, so I'm paused and nothing more happens until you do.
> **To continue:** type `/verify <task-id>`; after it passes I'll ask you to type `/ship <task-id>` the same way.
<!-- touchpoint:end -->

When you report that implementation is done (and again after `/ship`
completes), use this form; it replaces any freeform summary:

<!-- touchpoint:start report -->
> **Bottom line:** <where things stand in one plain sentence, and whether anything waits on you>
> **What I did:** <2-4 short lines: what now works or behaves differently, in user terms, not file names>
> **What you need to do:** <the next step and exact command, or "nothing">
> **Worth knowing:** <anything surprising, left open on purpose, or not proven, in plain words; or "nothing">
<!-- touchpoint:end -->

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

