---
name: tasks
description: List every open task in the registry — owner, class, phase, claims, and flags. Read-only. Exists so a human and claims-check/propagate see the same picture.
disable-model-invocation: true
argument-hint: []
---

You are running `/tasks`. No arguments.
Project root: the workspace root if `workspace.json` exists here, otherwise
this project. Scripts at `${CLAUDE_SKILL_DIR}/../../scripts/<name>` — hand
that to the shell verbatim, `../../` included; do **not** lexically
collapse it to `.claude/`, which is a symlink into the spine core checkout.

## 1. Pull the shared mainline first

**This is not optional** — the registry is whatever the shared mainline
says, and a stale local checkout re-opens exactly the visibility hole the
registry exists to close: registry-reading operations pull the shared
mainline before reading.

```
git -C <project root> pull --rebase --quiet
```

If this fails (no remote, offline, a real conflict), say so plainly —
"showing the last-known state as of `<local HEAD short sha>`, could not
pull — this list may be missing tasks opened elsewhere since then" — and
continue with what's on disk rather than refusing to show anything. This
is lower-stakes than `claims-check`'s own pull (§2.3, which gates an
approval) so a degraded read is the right failure mode here, not a halt.

## 2. Enumerate open tasks

Open = has a `work/<task-id>/state` file whose content is not `done`
(closed tasks vastly outnumber open ones in any real project's history —
open question 5: this is what keeps the scan sub-second regardless of how
many closed tasks accumulate, since it's one `cat` per task folder, not a
git-history walk):

```
for d in work/*/; do
  [[ -f "$d/state" ]] || continue
  s="$(cat "$d/state")"
  [[ "$s" == "done" ]] && continue
  echo "$d"
done
```

## 3. For each open task, read and report

- `work/<task-id>/owner` (one line)
- `work/<task-id>/class` (one line)
- `work/<task-id>/state` (the phase — this table's whole reason for
  existing is that this is the one place a human sees every open task's
  phase at a glance, not just their own)
- `work/<task-id>/claims.json`'s `predicted_touch`, `grounding_files`,
  `grounding_decisions`, `contracts` — summarize counts, not the full
  paths, unless the human asks for one task's detail
- `work/<task-id>/flags.json` — count of entries, and how many have
  `"acknowledged": false` (call these out specifically — an unacknowledged
  flag is blocking that task's next phase advance right now,
  `core/skills/task/SKILL.md`'s own enforcement point)

Present as a table: task id, owner, class, phase, claims (file/decision/
contract counts), flags (total / unacknowledged). Sort by task id (which
sorts by date first, per the `<YYYYMMDD>-<slug>` id shape) — oldest open
task first, since that's usually the one most likely to be blocking
something.

## 4. Nothing to editorialize

This skill never blocks, resolves, or recommends — it's the read-only view
`claims-check` and `propagate` also read state from, so a human seeing the
same table they'd see is the point — exists so humans and scripts see the
same picture. If the list is empty, say so plainly —
"no open tasks" is a real, useful answer, not nothing to report.
