---
name: update
description: Sync this project's spine install to the current checkout — re-runs setup to pick up new skills, agents, hooks, and rules, and shows what changed since the last sync. Run after pulling a new spine version.
disable-model-invocation: true
argument-hint: []
---

You are running `/update`.

**This is setup, not a task** — it re-syncs the project's skill/agent/
hook/rule symlinks to the current spine checkout, then reports what
changed. No `work/<task-id>/` folder, no floor, no class.

**Resolve the spine path first.** `${CLAUDE_SKILL_DIR}` may point to a
symlink (e.g. `.claude/skills/update`) rather than the real spine
checkout. Resolve it before using it:

```bash
# CLAUDE_SKILL_DIR resolves to <spine>/core/skills/update — three levels
# below the spine checkout root.
SPINE_ROOT=$(cd "$(readlink -f "${CLAUDE_SKILL_DIR}")/../../.." && pwd)
# e.g. /Users/you/spine/core/skills/update -> /Users/you/spine
```

Use `$SPINE_ROOT` for all paths below. If `readlink -f` is unavailable,
try `realpath` as a fallback; if both fail, ask the human for the spine
checkout path.

## 1. Re-sync symlinks

Run:

```
$SPINE_ROOT/core/scripts/setup --project <project-root>
```

where `<project-root>` is `$CLAUDE_PROJECT_DIR` (the directory this
session is running in). Capture and show the full output — it reports
symlink changes and any errors.

If setup exits non-zero, stop and show the error. Do not proceed to §2.

## 2. Show what changed

Show what the most recent `git pull` in the spine checkout brought in:

```
git -C $SPINE_ROOT log ORIG_HEAD..HEAD --oneline
```

If `ORIG_HEAD` doesn't exist or the range is nonsensical, fall back to
`git -C $SPINE_ROOT log -10 --oneline` and say that's what you did.

If the log is empty (project is already at HEAD), say:
> Already up to date — no new spine commits from the last pull.
> Symlinks re-synced.

If the log is non-empty, show it and give a one-sentence plain-language
summary of what categories of change landed (new skills, hook changes,
skill updates, bug fixes — read the commit messages to characterize them,
don't just dump the log).
