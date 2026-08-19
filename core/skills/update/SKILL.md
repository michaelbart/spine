---
name: update
description: Sync this project's spine install to the current checkout — re-runs setup to pick up new skills, agents, hooks, and rules; shows what changed since the last pin; and offers to bump the pin. Run after pulling a new spine version.
disable-model-invocation: true
argument-hint: [--bump-pin]
---

You are running `/update`. Scripts at
`${CLAUDE_SKILL_DIR}/../../scripts/`.

**This is setup, not a task** — it re-syncs the project's skill/agent/
hook/rule symlinks to the current spine checkout, then reports what
changed. No `work/<task-id>/` folder, no floor, no class.

## 1. Re-sync symlinks

Run:

```
${CLAUDE_SKILL_DIR}/../../scripts/setup --project <project-root>
```

where `<project-root>` is `$CLAUDE_PROJECT_DIR` (the directory this
session is running in). Capture and show the full output — it reports
version skew, symlink changes, and any errors.

If setup exits non-zero, stop and show the error. Do not proceed to §2.

## 2. Show what changed since the last pin

Read `<project-root>/.spine/core-pin.json`'s `sha` field (the spine
commit this project was last calibrated against). Run:

```
git -C <spine-checkout> log <pinned-sha>..HEAD --oneline
```

where `<spine-checkout>` is the spine directory resolved by setup (the
parent of this skill file's own directory: `${CLAUDE_SKILL_DIR}/../..`).

If the log is empty (project is already at HEAD), say:
> Already up to date — no new spine commits since the last pin.
> Symlinks re-synced.

If the log is non-empty, show it and give a one-sentence plain-language
summary of what categories of change landed (new skills, hook changes,
skill updates, bug fixes — read the commit messages to characterize them,
don't just dump the log).

## 3. Offer to bump the pin

If there are new commits (§2 log was non-empty) OR if `--bump-pin` was
passed:

> The pin in `.spine/core-pin.json` still points to `<old-sha>`. Bump it
> to `<new-HEAD-sha>` to record that this project has been updated?
> (Recommended after reviewing the changelog above.)

If the human confirms (or `--bump-pin` was passed without ambiguity):
edit `.spine/core-pin.json` — update the `sha` field to the current
spine HEAD. Commit:

```
chore: bump spine core pin to <short-sha>
```

No `Spine-Task:` trailer. If the human declines, leave the pin as-is and
note that the version-skew warning will continue to appear at the start
of each task until the pin is bumped.
