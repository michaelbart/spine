---
name: spine
description: Front desk — "where am I, what do I do next." Read-only. Checks install health, then reports the active task's status and single next action; or, when idle, the commands available; or, when the install is broken, exactly what to fix. The "lost? run this" command.
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

## 1. Are you in the spine core checkout, not a project?

If the current directory has `core/skills/` and `core/ADAPTER-CONTRACT.md`
(this repo's own shape) and no `.spine/` of its own, say so plainly: this is
the portable core, not a project that runs on it. Point them at `/bootstrap
--project <path>` (brand-new project) or `/adopt --project <path>` (existing
code), and stop. Nothing below applies.

## 2. Check install health first — always

Before reporting anything else, run the cheap health check — the same one
`core/skills/task/SKILL.md`'s step 0 runs, for the same reason: a core skew or
a missing enforcement layer matters *most* when there's active work, not
least.

```
${CLAUDE_SKILL_DIR}/../../scripts/setup --check --project <project root>
```

Interpret its outcome with the three-way discipline every skill uses
(`core/skills/task/SKILL.md`'s tooling-gap section) — "ran and passed," "ran
and reported a problem," and "could not run at all":

- **Not installed** (exit 2 / "capabilities.json not found" / "no
  `.claude/hook-guard`") — this project isn't set up (or its enforcement layer
  is missing). Explain in a sentence or two that spine installs by symlink
  from a core checkout, and give the exact command: a second engineer on an
  already-installed project runs `<spine>/core/scripts/setup --project
  <project root>`; a brand-new project needs `/bootstrap` or `/adopt` from a
  session in the spine repo. Then **stop** — there's no task state to report.
- **Core skew** ("CORE VERSION SKEW", warn or strict) **or hook-guard
  differs/stale** — the install works but this machine's core is out of sync.
  **Relay the exact message and the fix command it printed, verbatim** (`git
  pull` in the spine checkout, then `setup`); don't soften a strict skew into a
  warning. This is a warning, not a stop — surface it, then **continue** to the
  report below, because the engineer still needs to know their task status
  (and needs the skew warning precisely because they may be about to ship).
- **Clean** (exit 0, no skew, guard present) — say nothing about it. A healthy
  install needs no announcement. Continue.
- **Could not run at all** (the call was blocked/denied/errored before the
  script's own logic) — say so plainly ("couldn't run the install check, so I
  can't confirm this machine's core is in sync") rather than reporting a clean
  bill. Low-stakes and read-only, so continue to the report below — just don't
  claim health you couldn't verify.

## 3. Now report: idle, or a task in progress

If `.spine/current-task` does **not** exist -> idle, show the menu (§3.1). If
it **does** exist -> a task is in progress, show the status (§3.2).

### 3.1 Idle — the menu

Give a short, plain menu — one sentence each, the starting move first. List
only commands that exist in this install (they're symlinked under
`.claude/skills/`; don't advertise one that isn't there):

- **`/task <description>`** — the way to start a piece of work: classify ->
  research -> plan (you approve it) -> implement -> verify -> ship. *This is
  the one to reach for first.*
- **`/visualize`** — open the project dashboard (timeline, decisions,
  capabilities, drift) in a browser.
- **`/tasks`** — list every open task and its phase.
- **`/costs`** — what spine is costing, drift first.
- **`/design`**, **`/ratchet`**, **`/remap`** — foundational design, converting
  a recurring friction into a check, and regenerating the map; mention these
  only briefly, as "also available."

Keep it to what a confused engineer needs: the front door and the two or three
things they'd want next. Don't reproduce the whole README.

### 3.2 In progress — status and the single next action

Read the active task's real state — never from memory, always from disk (some
of these files may be absent; a missing `deviations.md` just means no
deviations, not an error):

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
  Lead with it: quote what changed and say it must be acknowledged before the
  task can move on. This outranks the phase-based next action.
- **Any `open` halt-tier deviation** — the task is waiting on a human decision;
  point them at `work/<task-id>/deviations.md`.
- Otherwise, by phase: `research`/`plan` — spine is grounding or drafting; if a
  `work/<task-id>/plan.md` exists and the phase is `plan`, the next action is to
  review and approve it. `implement` — work is underway; nothing for them unless
  a deviation opens. `verify` — if `work/<task-id>/verify.md` does not exist yet,
  the next action is to run `/verify <task-id>` (it can't be auto-invoked); if it
  exists and its `Result:` line reads `PASS`, the next action is to run `/ship
  <task-id>`; if `FAIL`, the task needs fixes and a re-run of `/verify`. `ship` —
  the next action is to run `/ship <task-id>`.

If a `workspace.json` is present, note that this is a workspace-root task and
name the member repos it touches (from the plan's `## Ship order` or `##
Predicted touch`) so they know its scope isn't a single repo.

Resuming is `/task`'s job, not yours — if they want to continue the work, the
next action you name (or `/task` with no argument, which offers to resume) is
how, but `/spine` itself never advances anything.

## 4. Never editorialize

Like `/tasks`, this skill reports and stops. It never blocks, never fixes,
never decides. "No active task — here's how to start" and "you're mid-verify
and it passed, run `/ship <id>` next" are both complete, useful answers. If the
install is healthy and idle, the menu *is* the whole output — don't manufacture
a status for a task that doesn't exist.
