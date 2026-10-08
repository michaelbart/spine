# Task — milestone and auto-continue (loaded on demand)

Loaded from SKILL.md when: the description is empty or `--milestone <id>` is given.

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
that already looked complete. Its one-line output branches five ways:

- **`TBD <milestone-id> <description>`** — propose it and stop: "Continue
  with `<milestone-id>`'s next task: `<description>`?" — before doing
  anything else. This is a real touchpoint, not a
  courtesy notice: nothing has been created yet (no task folder), so it's
  the cheapest possible point to catch a wrong guess, one keystroke against retyping the whole
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
- **`DISPOSITION <milestone-id> <task-id>`** — a planned task in that milestone
  was abandoned, so it can't finish as written. Ask before anything else, with
  `AskUserQuestion`:

  <!-- touchpoint:start confirm -->
  > **Replace the abandoned task "<its description>" in `<milestone title>`, or drop it from the plan?** It was set aside on purpose, and the milestone can't finish until you choose.
  > **Replace it** (recommended) — I queue it again as a fresh task. **Drop it** — I remove it from the plan and note why.
  <!-- touchpoint:end -->

  Replace: set that entry in `work/<milestone-id>/milestone.md` back to `TBD`
  (keep its description) and carry on as the `TBD` case. Drop: delete the entry,
  add the description and the reason (second line of `work/<task-id>/abandoned`)
  under `## Known gaps for future member tasks`, and re-run the script.
- **`NONE`** — no milestone exists yet, or every one found is already
  complete. Fall through to asking for a description, saying briefly why
  auto-continue didn't fire.
- If the script could not run at all, this is a tooling gap — say so
  plainly to the human ("couldn't check for a queued milestone task") and
  fall through to asking for a description; don't treat a script that
  couldn't execute as "nothing queued."

**If `--milestone <id>` is given:** look for `work/<id>/milestone.md`.
**If it does not exist, the milestone is new — before creating it, stop
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
`work/<id>/milestone.md`, seeded from
`core/templates/milestone.md` plus whatever gap entries were carried
forward (bump `next-gap-id` past the highest carried id). **If
`work/<id>/milestone.md` already exists, none of this applies** — either
`/roadmap` already ran and did it, or an earlier task in this same milestone
already did.

Read `work/<id>/milestone.md` (design-stage extension,
`core/templates/milestone.md`) before classifying — its `## Member tasks`
list and `## Capability targets` both become planning context for every
phase below. Once
this task's ID is generated (step 1), replace this milestone's first
still-`TBD` member-task line with the real task ID (`Edit` on
`milestone.md` — this is bookkeeping, not a phase artifact). **Do
this before writing `work/<task-id>/state` in step 1, not after** —
`phase-gate` only restricts `Edit`/`Write` to a task's own
`work/<task-id>/` once that task has a `state` file reading `research` or
`plan`; with no `state` file written yet, this edit is simply outside the
hook's gating window, not something the hook has to carve out a special
case for. Record `work/<task-id>/milestone` = `<id>`, one line, so `/ship`
(final member task's own done-definition check) and a resumed session both
know this task belongs to a milestone without re-parsing `$ARGUMENTS`.
