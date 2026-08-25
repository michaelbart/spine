# Implementation plan: per-terminal worktree isolation for `/task`

Status: proposed, not built. Written 2026-08-25 after confirming the gap —
see "What exists today" below. Trigger chosen: **auto-detect** (`/task`
notices another task is already active in this working tree and offers a
worktree for the new one), not an explicit `--worktree` flag and not
"always worktree."

## The problem

Today, a project has exactly one live task per working directory:
`.spine/current-task` is a single line, and `core/hooks/phase-gate` /
`path-escalate` read it as *the* active task for whatever directory
`$CLAUDE_PROJECT_DIR` resolves to. If an engineer opens a second terminal
in the same repo checkout and runs `/task` on something unrelated, `/task`'s
own "Resuming" branch (`core/skills/task/SKILL.md`, the "Resuming:"
paragraph right after the registry-sync section) treats the existing
`current-task` as the thing to resume — there is no path that starts a
second, independent task in the same checkout. Even if there were, both
terminals would be editing the same files on the same branch: the first
task's uncommitted implementation work would be sitting in the same
working tree the second task starts implementing into.

Nothing in spine builds a feature-branch-per-task or worktree-per-task
convention today. The only existing worktree usage is
`core/agents/falsifier.md` (`isolation: worktree`), which is a short-lived,
throwaway sandbox for one subagent's mutation-testing probe during
`/verify` — unrelated to a human running two tasks at once.

## What exists today that this plan reuses

Claude Code itself already has a native git-worktree primitive:
`EnterWorktree` / `ExitWorktree`. `EnterWorktree` creates a worktree under
`.claude/worktrees/` on a new branch (base ref per the `worktree.baseRef`
setting: `fresh` = `origin/<default-branch>`, `head` = current local HEAD)
and switches the session's own working directory into it; `ExitWorktree`
returns to the original directory and can keep or remove the worktree.
This repo already has a (currently empty) `.claude/worktrees/` directory
from past use. This plan is "teach `/task` to call these at the right
moment," not "build git-worktree plumbing from scratch."

## The real gap this plan has to close

A plain `git worktree add` only checks out **tracked** files. Per
`core/scripts/setup`:

- `.claude/skills/`, `.claude/agents/`, `.claude/rules/`, `.claude/hooks`,
  and `.claude/settings.local.json` are **gitignored** — they're symlinks
  (or, for settings.local.json, a machine-local file) `setup` generates
  pointing at this machine's own spine core checkout
  (`core/scripts/setup:216-229, 279-283`).
- `.claude/settings.json` (the file that wires
  `${CLAUDE_PROJECT_DIR}/.claude/hook-guard phase-gate` etc. into
  PreToolUse — `core/scripts/setup:257-259`) and `.spine/adapters/*`,
  `.spine/protected-paths.conf`, `.spine/profile.json`,
  `.spine/capabilities.json`, `work/` **are** tracked/committed real
  content, so a fresh worktree checkout gets those automatically.

So a worktree created by a bare `EnterWorktree` call would have hook
*wiring* (settings.json) but no hook *targets* (the symlinks) — every
`phase-gate`/`path-escalate`/`dep-gate` invocation would fail to resolve,
and every `/task`-referenced script path
(`${CLAUDE_SKILL_DIR}/../../scripts/...`) would break because
`.claude/skills/task` itself wouldn't exist. This is the one real piece of
new plumbing: **run `core/scripts/setup --project <worktree-path>`
immediately after entering the worktree, before any other spine logic runs
there.** `setup` is documented idempotent and safe to re-run
(`core/scripts/setup:53`, "--check: read-only" implies the non-check path
is the normal re-run path); it requires `.spine/capabilities.json` to
already exist (`core/scripts/setup:191-192`), which the worktree's tracked
checkout already satisfies.

## Design

### 1. Detection (new step in `/task`, before Step 0)

Right after the existing empty-description auto-continue check and before
"Step 0 — core version check," add: if `.spine/current-task` already
exists **and** a real (non-empty) description was given this invocation
(i.e., this is not itself a resume), read that file's `state` and
`class`, and ask the human plainly:

> "`<existing-task-id>` is active here (phase `<state>`). Start
> `<new description>` in a separate worktree instead of interrupting it,
> or resume/switch to `<existing-task-id>` here instead?"

Three answers, all explicit — never silently pick one:

- **New worktree** — proceed to §2 below.
- **Resume/switch to the existing task** — fall through to today's
  existing "Resuming" behavior, unchanged.
- **Something else the human types** — follow it; don't re-guess.

Guard against nesting: skip this whole check (proceed exactly as today)
if the current working directory is already a spine-created worktree
(cheapest signal: cwd is under a path containing `/.claude/worktrees/`).
A worktree task should never itself offer to spin up a worktree.

### 2. Entering the worktree

```
EnterWorktree({ name: <slug derived from the description> })
```

This switches the session's cwd. Immediately after, before anything else:

```
core/scripts/setup --project <new worktree path>
```

(resolve the spine core checkout path the same way `/task`'s own header
resolves `${CLAUDE_SKILL_DIR}/../../scripts/`, so this works whether spine
was installed via symlink or direct checkout). Treat a `setup` failure
here as a hard stop, not a tooling-gap-and-continue — running any
`/task` logic in a worktree with broken hook wiring would silently defeat
`phase-gate`/`path-escalate` for that entire task.

Then **proceed exactly like a normal fresh `/task <description>`
invocation** from this new cwd — Step 0, classify, research, plan, etc.,
all unchanged. `.spine/current-task` in the *new* worktree is independent
of the original checkout's; the two tasks never see each other's phase
files because they're now different directories.

### 3. What does NOT need to change

- `phase-gate` / `path-escalate` — both already resolve everything off
  `$CLAUDE_PROJECT_DIR`/`$(pwd)`, never a hardcoded repo root, so they work
  unmodified once the worktree has its own `.claude/hooks` symlink.
- `core/skills/task/SKILL.md`'s phase logic, ledger, claims.json,
  flags.json — all task-id-scoped and path-relative; no changes needed
  beyond the new detection step in §1.
- The falsifier's own `isolation: worktree` mechanism — orthogonal, keeps
  working as-is inside whichever worktree `/verify` runs in.

### 4. Exit / cleanup, at `/ship`'s close-out (§6)

`core/skills/ship/SKILL.md` §6 already removes `.spine/current-task` when
a task reaches `done`. Add: if this task's cwd is a spine-created
worktree (same cheapest signal as §1's guard), ask the human once:
"This task ran in worktree `<path>` — remove it now, or keep it (e.g.
you're still watching the PR)?" Default the *offered* choice to keep
(`ExitWorktree({action: "keep"})`) if the human doesn't answer, since
removal is the harder-to-reverse of the two and a clean ship should have
no uncommitted changes left to lose either way. Never call `ExitWorktree`
with `discard_changes: true` automatically — only on explicit human
confirmation, per the tool's own contract.

### 5. Known limitation to disclose (not block on)

`EnterWorktree` puts each worktree on its **own new branch** — git can't
check the same branch out in two worktrees at once. That means
`registry-sync` (which pushes `work/<task-id>/` commits to "whatever
branch is currently checked out," documented in `core/scripts/
registry-sync:10-19` as assuming that's the shared mainline) will push a
worktree task's registry entries to that task's own branch, not to the
project's default branch — so `claims-check`, `gaps-report`, and a
colleague's own session won't see that task's `work/` folder until its
branch merges. For the actual use case here (one engineer, one machine,
two terminals) this is harmless — nothing is hidden from *that* engineer's
own next `/task` invocation, since both worktrees share the same local
`.git`. It only matters for the cross-engineer coordination story, which
this feature isn't targeting. Disclose it in `docs/tradeoffs.md` next to
the existing `auto`-autonomy disclosure, the same way that tradeoff was
handled — don't silently ship the gap undocumented.

## Open questions to resolve before/while building

1. **Worktree naming.** `EnterWorktree`'s `name` param needs a slug at
   detection time (§1), before the task-id's own `<YYYYMMDD>-<kebab-slug>`
   is generated later in Classify. Either compute the slug once, early,
   and reuse it for both, or accept the worktree directory name and the
   eventual task-id being independently-derived (minor cosmetic
   mismatch, no functional cost).
2. **Multi-repo (`workspace.json`) interaction** — out of scope for this
   plan's first pass. A workspace's member repos are separate git repos;
   worktree-per-task there would need per-repo `EnterWorktree` calls
   coordinated at the workspace root. Flag as unexplored, don't guess at
   a design until someone actually hits it.
3. **Whether to also add an explicit `--worktree` escape hatch** (start
   in a worktree even when nothing else is active) — low cost to add once
   §2's plumbing exists, since it's the same code path minus the §1
   detection gate. Worth doing as a fast follow, not blocking the
   auto-detect path.

## Suggested build order

1. `core/scripts/setup --project <path>` re-run behavior when pointed at
   a fresh worktree — verify by hand (`git worktree add` + run `setup`)
   that symlinks, `settings.local.json`, and hook wiring all come up
   correct, before touching `/task` itself.
2. Wire §1 (detection) + §2 (enter + setup + proceed) into
   `core/skills/task/SKILL.md`.
3. Wire §4 (exit/cleanup) into `core/skills/ship/SKILL.md` §6.
4. Disclose the registry-visibility limitation (§5) in
   `docs/tradeoffs.md`.
5. Dry-run: two terminals, same repo, second one triggers the new prompt,
   confirm the two tasks' `work/<task-id>/` folders, `.spine/current-task`
   files, and implementation edits stay fully independent end to end
   through `/ship`.

This whole feature is itself exactly the kind of change spine's own
`/task` should run through in an *installed* project — but this repo is
the portable core, not an installed project (its own `.gitignore` says so
explicitly), so building it happens as a normal `/task` in whichever
project actually consumes this core, or by hand here with the same
research → plan → implement → verify discipline spine prescribes
elsewhere.
