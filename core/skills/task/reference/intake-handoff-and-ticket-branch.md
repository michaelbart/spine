# Task — intake handoff and ticket branch (loaded on demand)

Loaded from SKILL.md when: `.spine/current-intake` exists.

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
Resuming check above already resolved that no other task is active in
this directory — the one place a branch switch can't collide
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
  ticket).
- Otherwise a remote-tracking one exists (`git rev-parse --verify --quiet
  refs/remotes/<remote>/<branch>`, `<remote>` read the same way as below) —
  `git checkout -b <branch> <remote>/<branch>`.
- Otherwise, create it fresh from current HEAD: `git checkout -b <branch>`.
  Then, if the branch you were just on has a configured remote
  (`git config branch.<previous-branch>.remote`), immediately `git push -u
  <remote> <branch>` — this is what gives `open-pr`'s "uses the
  current branch when `SPINE_PR_HEAD` is unset" (`core/ADAPTER-CONTRACT.md`
  §3.5) a real head branch to open a PR from, instead of silently assuming
  the human already branched by hand. No remote configured on the previous
  branch: skip the push, stay local (a normal solo/single-machine project, not
  a gap). A push failure here (network, permissions) is non-fatal — note it
  and continue on the local branch; `/ship`'s push retries it naturally once
  the human resolves it by hand.

One more thing rides along in the setup step below: for a direct
`/task` with no handoff, write the autonomy the human just chose in the step
above (Class 2 / a downgraded task ⇒ `guided`); and if
`class_below_recommended` is true, add a `notes.md` line recording the
downgrade — the downgrade stays visible without consuming a circuit-breaker
slot (it is a classification choice, not a plan-vs-reality deviation, so it
is **not** a `deviations.md` record). Then **delete `.spine/current-intake`**
and continue to §2. The class-suggestion list below is
only for a `/task` invoked directly, with no intake handoff.
