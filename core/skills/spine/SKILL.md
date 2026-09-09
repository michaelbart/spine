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
`${CLAUDE_SKILL_DIR}` is a placeholder you expand to this skill's own
directory; hand the resulting path — including the `../../` — to the shell
verbatim. Do **not** lexically collapse `skills/spine/../..` to `.claude/`:
`.claude/skills/spine` is a symlink into the spine core checkout, so the
shell must resolve `../../` against the symlink's real target
(`<spine>/core/...`). Collapsing it as text yields a nonexistent
`.claude/scripts/...` path and a "no such file" error.

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

## 3. Handoff staleness (read-only, additive)

A real, recurring miss: a product-spec/design-handoff document gets saved
at `docs/product-spec.md` and either never gets reconciled through
`/design --handoff` at all, or gets *changed* (a supplementary delivery,
a revision) after `/design` already consumed an earlier version of it —
and nothing notices, because "did the handoff change" was previously
something only a human remembering to check would catch. This check
closes that mechanically, the same way the install-health check above
catches core skew instead of hoping someone notices:

1. If `docs/product-spec.md` doesn't exist, skip this section silently —
   nothing to check.
2. If `docs/decisions/D-*.md` don't exist yet either, skip silently too —
   `/design` hasn't run yet at all, and its own §0 preflight already
   auto-detects `docs/product-spec.md` and offers it as `--handoff` the
   moment it does run; nothing for `/spine` to add before then.
3. Otherwise, decisions exist — `/design` has run at least once. Read
   `.spine/handoff-consumed-sha` if it exists (line 1 the path it stamped,
   line 2 the hash). Compute `git hash-object docs/product-spec.md`'s
   current value.
   - **No stamp file at all** — decisions exist but this document was
     never consumed via `--handoff`. Say so plainly: *"`docs/
     product-spec.md` exists but doesn't look like it's been reconciled
     through `/design --handoff` — worth checking whether it should be."*
   - **Stamp exists, path matches, hash differs** — the document changed
     since `/design` last read it. Say so plainly: *"`docs/product-spec.md`
     has changed since `/design` last reconciled it — consider `/design
     --handoff docs/product-spec.md` before planning further work from
     it."*
   - **Stamp exists and hash matches** — current as of the last `/design`
     run; say nothing, same as a clean install-health check.
   - **Stamp names a different path** — a second, distinct handoff
     document exists at `docs/product-spec.md` that was never stamped at
     all under its own name; treat this the same as "no stamp file,"
     scoped to `docs/product-spec.md` specifically.

Never resolve this yourself — report it and move on, same read-only
stance as every other check in this skill. If `git hash-object` couldn't
run at all, say so plainly and skip this check rather than guessing.

## 4. Wayfinder map in progress (read-only, additive)

If `.spine/current-wayfinder` exists, report it before anything else
below — independent of whether a task is also active, the same way the
workspace member-repo milestone check (§5.1) is additive to the
workspace-root callout rather than replacing it:

```
${CLAUDE_SKILL_DIR}/../../scripts/wayfinder-frontier --map <map-id> --project <project root>
```

Report the map id, its `## Destination` one-liner (from `work/wayfinder/
<map-id>/map.md`), and the frontier script's outcome in plain words —
"N tickets open, ready to work: T2, T4" for `FRONTIER`, "every ticket
resolved — run `/wayfinder` to write it into `docs/vision.md`" for
`CLEARED`, or "stalled — every open ticket has an unresolved blocker, run
`/wayfinder` to sort it out" for `STALLED`. Never resolve or advance
anything here — same read-only stance as every other report this skill
gives. If the script could not run at all, say so plainly and skip this
callout rather than guessing.

## 4a. Known issues (read-only, additive)

`docs/known-issues.md` (`core/skills/note-issue/SKILL.md`) is a
low-ceremony ledger for observations logged outside any active task or
ticket — structurally separate from any milestone's own provenance-locked
Known-gaps section. Nothing else surfaces it between `/roadmap` runs, so
without this it can rot invisibly. Surface the open count, same additive,
read-only stance as every other check in this skill:

```
${CLAUDE_SKILL_DIR}/../../scripts/issue-ledger count --open --project <project root>
```

Non-zero: one line — "N open known issue(s) logged — `/roadmap` absorbs
them into a milestone (or an explicit decline) next time it runs." Zero,
or `docs/known-issues.md` doesn't exist yet: say nothing, same as a clean
install-health check. If the script could not run at all, say so plainly
("couldn't check known-issues.md") rather than reporting a clean bill.

## 5. Now report: idle, or a task in progress

If `.spine/current-task` does **not** exist -> idle, show the menu (§5.1). If
it **does** exist -> a task is in progress, show the status (§5.2).

### 5.1 Idle — milestone check, then the menu

"Idle" only means no task is *currently open* — it doesn't mean there's
nothing in flight. Before the generic menu, check whether the project is
mid-milestone, read-only, exactly like every other check in this skill:

1. Glob `work/M*/milestone.md`. None exist -> this project isn't using
   milestones; skip straight to the menu below.
2. For each one found, read its `## Member tasks` numbered list. Each entry
   names either a real task id or the literal `TBD`.
3. A milestone is **complete** when every member entry is a real task id and
   each of those tasks' `work/<task-id>/state` reads `done`. Find the
   lowest-numbered milestone that is *not* complete — that's the current one.
   If every milestone is complete, say so in one line. Then check
   `docs/vision.md` — if it exists, read its planned milestone list and
   find the first milestone title that has no corresponding
   `work/M*/milestone.md` yet; surface it as the next step: *"Per
   `docs/vision.md`, the next planned milestone is M<n>: <title> — run
   `/roadmap` to plan it, or `/task <description> --milestone M<n>` to
   start the first task directly."* If `docs/vision.md` is absent or
   every listed milestone already has a folder, fall through to the menu.
4. Report the current milestone plainly: its id and title (the file's `#`
   heading), and progress as "`<n>` of `<total>` member tasks done." Then name
   **the single next action**:
   - If the first non-done member entry is still `TBD`, quote that member
     task's own description from `milestone.md` and give the exact command to
     open it: `` /task <description> --milestone <id> ``.
   - If it's a real task id whose `state` isn't `done`, name that task and its
     phase and point at `/task` (no argument) to resume it, or the
     phase-appropriate next command per §5.2's transition table.
5. This is a report, not a gate (§5 applies here too) — it never opens the
   task itself, only names the command that would.

#### Member-repo milestone check (workspace only)

If `workspace.json` exists at the project root **and** no `.spine/current-task`
exists at the workspace root (i.e., the workspace itself is idle), also scan
each member repo for mid-milestone state. This is additive — the workspace's
own milestone callout (if any) still appears first; member-repo callouts follow
below it.

For each entry in `workspace.json`'s `repos` array, take its `path` field and:

1. Glob `<path>/work/M*/milestone.md`. If none exist (or `.spine/` is absent),
   skip this repo silently.
2. Apply the same milestone-progress logic as steps 2–4 above, but **against
   the member repo's own task files** (`<path>/work/<task-id>/state`, etc.).
3. Find the lowest-numbered incomplete milestone and report it clearly
   attributed to the member repo. Use this format:

   > In member repo `<repo-name>` (M*n*, *x* of *total* tasks done): next task
   > is TBD — open it here with `/task <description> --milestone M*n*`

   or, if the next task is a real-but-open task id:

   > In member repo `<repo-name>` (M*n*, *x* of *total* tasks done): next task
   > is `<task-id>` (*phase*) — switch to that repo and run `/task` to resume.

   `<repo-name>` is the last path segment of `path` (e.g. `waitlist` from
   `../g1-svc-api/modules/waitlist`).

4. If every milestone in the member repo is complete, skip it silently.

This check is read-only and purely additive. Errors accessing a member repo
(path not found, no `.spine/`, no `work/`) are silently skipped — never
surfaced as errors to the engineer.

After the milestone callout (or immediately, if there is none), give the
short, plain menu — one sentence each, the starting move first. List
only commands that exist in this install (they're symlinked under
`.claude/skills/`; don't advertise one that isn't there):

- **`/intake <ticket>`** — the front door for ticketed work: fetch the ticket,
  size it against the real code, and route it into the right flow. *Reach for
  this first when you pick up a ticket.*
- **`/task <description>`** — start work directly, without a ticket: classify ->
  research -> plan (you approve it) -> implement -> verify -> ship.
- **`/design`**, **`/wayfinder`**, **`/prototype`**, **`/ratchet`**,
  **`/remap`**, **`/prompts`** — foundational design; charting a large foggy
  effort into a map of decision tickets; a disposable spike to settle a
  visual/behavioral question; converting a recurring friction into a
  check; regenerating the map; and printing a ready-to-paste handoff
  prompt (product spec, feature handoff, or UI handoff) for use in
  another session; mention these only briefly, as "also available."

Keep it to what a confused engineer needs: the front door and the two or three
things they'd want next. Don't reproduce the whole README.

### 5.2 In progress — status and the single next action

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

## 6. Never editorialize

This skill reports and stops. It never blocks, never fixes,
never decides. "No active task — here's how to start," "you're mid-M1, next
member task is X," and "you're mid-verify and it passed, run `/ship <id>` next"
are all complete, useful answers. If the install is healthy, idle, and no
milestone is in flight, the menu *is* the whole output — don't manufacture a
status for a task or milestone that doesn't exist.
