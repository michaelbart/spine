---
name: spine
description: Front desk — "where am I, what do I do next." Read-only. Reports the active task's status and single next action; or, when idle, the commands available; or, when the install is broken, exactly what to fix. The "lost? run this" command.
disable-model-invocation: true
argument-hint: []
---

You are running `/spine`, the front desk. Its whole job is that an engineer
never has to remember how spine works: it tells them where they are and what
to type next. **It is strictly read-only — it never edits a file, advances a
phase, resolves a flag, runs a gate, or recommends a decision. It reports.**
No arguments.

Project root: the workspace root if `workspace.json` exists here, otherwise
this project. Scripts live at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`.

## 1. First, figure out which of four states you're in

Check, in this order, and act on the first that matches:

**A — you're in the spine core checkout itself, not an installed project.**
If the current directory has `core/skills/` and `core/ADAPTER-CONTRACT.md`
(this repo's own shape) and no `.spine/` of its own, say so plainly: this is
the portable core, not a project that runs on it. Point them at
`/bootstrap --project <path>` (brand-new project) or `/adopt --project
<path>` (existing code), and stop. Nothing below applies.

**B — a project that hasn't been installed (or whose install is broken).**
Run the cheap health check and let it tell you:

```
${CLAUDE_SKILL_DIR}/../../scripts/setup --check --project <project root>
```

Interpret its outcome using the same three-way discipline every skill uses
(`core/skills/task/SKILL.md`'s tooling-gap section) — "ran and passed," "ran
and reported a problem," and "could not run at all":

- Exit 2 / "project isn't installed" / "no `.claude/hook-guard`" — this
  project isn't set up yet (or its enforcement layer is missing). Explain in
  one or two sentences that spine installs by symlink from a core checkout,
  and give the exact command: a second engineer on an already-installed
  project runs `<spine>/core/scripts/setup --project <project root>`; a brand
  new project needs `/bootstrap` or `/adopt` from a session in the spine repo.
  Then stop — there's no task state to report yet.
- "CORE VERSION SKEW" (warn or strict) or "hook-guard … differs / stale" —
  the install works but this machine's core is out of sync. Relay the exact
  message and the fix command it printed (`git pull` in the spine checkout,
  then `setup`), verbatim. Don't soften a strict skew into a warning.
- **Could not run at all** (the call was blocked/denied/errored before the
  script's own logic) — say that plainly ("couldn't run the install check,
  so I can't confirm this machine's core is in sync") rather than reporting a
  clean bill. This is low-stakes and read-only, so continue to the state
  report below on what's on disk — just don't claim health you couldn't
  verify.

If the check came back clean (exit 0, no skew, guard present), don't narrate
it — a clean install needs no announcement. Move on.

**C — installed, nothing in progress.** No `.spine/current-task` file. Show
the menu (§2).

**D — installed, a task is in progress.** `.spine/current-task` exists. Show
the status (§3).

## 2. Idle — the menu

Read `.spine/current-task`'s absence as "no active task." Give a short,
plain menu — one sentence each, the starting move first. List only commands
that exist in this install (they're symlinked under `.claude/skills/`; don't
advertise one that isn't there):

- **`/task <description>`** — the way to start a piece of work: classify →
  research → plan (you approve it) → implement → verify → ship. *This is the
  one to reach for first.*
- **`/visualize`** — open the project dashboard (timeline, decisions,
  capabilities, drift) in a browser.
- **`/tasks`** — list every open task and its phase.
- **`/costs`** — what spine is costing, drift first.
- **`/design`**, **`/ratchet`**, **`/remap`** — foundational design,
  converting a recurring friction into a check, and regenerating the map;
  mention these only briefly, as "also available."

Keep it to what a confused engineer needs: the front door and the two or
three things they'd want next. Don't reproduce the whole README.

## 3. In progress — status and the single next action

Read the active task's real state — never from memory, always from disk:

- `.spine/current-task` — the task id.
- `work/<task-id>/state` — the phase (`research` / `plan` / `implement` /
  `verify` / `ship`).
- `work/<task-id>/class` — `0` / `1` / `2`.
- `work/<task-id>/owner` — who opened it.
- `work/<task-id>/milestone` — the milestone id, if this task belongs to one.
- `work/<task-id>/flags.json` — count entries, and how many are
  `"acknowledged": false`.
- `work/<task-id>/deviations.md` — count records whose status is `open`.

Report, in plain words: the task id and title, its class, the current phase,
and then **the one thing to do next** — because "what do I type now" is the
question this command exists to answer. Derive the next action from the phase
and the blockers, mirroring `core/skills/task/SKILL.md`'s own transitions:

- **Any unacknowledged flag** — this blocks the next phase advance right now.
  Lead with it: quote what changed and tell them it must be acknowledged
  before the task can move on. This outranks the phase-based next action.
- **Any `open` halt-tier deviation** — the task is waiting on a human
  decision; point them at `work/<task-id>/deviations.md`.
- Otherwise, by phase: `research`/`plan` — spine is grounding or drafting; if
  a `work/<task-id>/plan.md` exists and the phase is `plan`, the next action
  is to review and approve it. `implement` — work is underway; nothing for
  them unless a deviation opens. `verify` — the next action is to run
  `/verify <task-id>` themselves (it can't be auto-invoked), or if
  `work/<task-id>/verify.md` already exists, read its `Result:` line.
  `ship` — the next action is to run `/ship <task-id>`.

If a `workspace.json` is present, note that this is a workspace-root task and
name the member repos it touches (from the plan's `## Ship order` or
`## Predicted touch`) so they know its scope isn't a single repo.

Resuming is `/task`'s job, not yours — if they want to continue the work,
the next action you name (or `/task` with no argument, which offers to
resume) is how, but `/spine` itself never advances anything.

## 4. Never editorialize

Like `/tasks`, this skill reports and stops. It never blocks, never fixes,
never decides. "No active task — here's how to start" and "you're mid-verify,
run `/verify <id>` next" are both complete, useful answers. If the install is
healthy and idle, the menu *is* the whole output — don't manufacture a status
for a task that doesn't exist.
