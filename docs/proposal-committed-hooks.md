# Proposal: commit `.claude/hooks/` into the project instead of symlinking it

Status: **proposed, not designed, not built.** Written up after a support
session that hit several real bugs in the symlink+pin model (a `jq` crash on
first `setup`, a backwards-direction skew message) and, while fixing them,
kept running into the same root cause: the actual enforcement code
(`.claude/hooks/*`) doesn't exist in a fresh clone until a human runs a
separate, easy-to-forget step, and Claude Code's own hook runner silently
no-ops a `PreToolUse` command pointing at a path that isn't there yet. This
is a design note to come back to, not a plan to execute — see the
recommendation at the bottom.

## The current model, and what it costs

Today, `core/scripts/setup` creates `.claude/skills/*`, `.claude/agents/*`,
`.claude/rules/*`, and `.claude/hooks` in an installed project as symlinks
into whatever spine checkout exists on that machine. Nothing spine-related
is actually committed as real content — the project's own `.claude/
settings.json` only wires `${CLAUDE_PROJECT_DIR}/.claude/hook-guard <name>`,
a tiny shim that execs into the real (symlinked, gitignored) hook if it
exists, or fails closed with a fix command if it doesn't.

This buys one real thing: pull spine once, and every project on that
machine picks up the improvement immediately, with no per-project commit.
That's the entire value proposition, and it's a good one.

It also creates an entire category of problem that doesn't need to exist:

- **`hook-guard` itself** — a whole file, committed and regenerated on every
  `setup` run, whose only job is "notice the real hook is missing and fail
  loudly instead of silently." It exists *only* because the real hook can
  be silently absent. Remove the possibility, and this file has no reason
  to exist.
- **The core-pin/skew-check machinery** (`.spine/core-pin.json`,
  `setup --check`, the `mismatch-warn`/`mismatch-strict` messaging we just
  fixed for direction) — exists to detect and surface that two engineers on
  the same project commit might be running *different code* for the same
  hook, because each one's `.claude/hooks` resolves to their own,
  independently-versioned spine clone. If the hook were just a committed
  file, this entire mismatch can't happen — whatever's checked out *is* the
  enforcement, for everyone, unconditionally.
- **Every bug we fixed this session** (the `jq null` crash stripping a
  nonexistent `permissions.allow`, the backwards skew-direction message)
  exists inside that machinery. Removing the machinery removes the surface
  those bugs live on, not just today's two instances of it.

## The proposal

Split distribution by risk, instead of treating everything spine installs
the same way:

- **`.claude/hooks/*` — commit as real files**, not symlinks. They gate
  `Edit`/`Write`/`Bash` directly; a stale or missing one is a security-shaped
  failure (silent fail-open), not a UX inconvenience. Whatever's in the
  repo at a given commit is unconditionally what enforces for everyone who
  has that commit checked out — no separate "did you run setup" step, no
  pin, no skew, no `hook-guard` shim needed at all.
- **`.claude/skills/*`, `.claude/agents/*`, `.claude/rules/*` — keep
  symlinked**, unchanged. These are prompt content, not enforcement; a
  stale skill costs a slightly-outdated instruction, not a bypassed gate.
  The "update once, every project benefits on next pull" property is worth
  keeping exactly where it is today.

## What this eliminates

- `core/templates/hook-guard` and the "regenerate it every setup run"
  logic in `core/scripts/setup`.
- The fresh-clone-enforcement problem `hook-guard` was built to solve
  (Extension C Phase C) — there's no window where a committed
  `settings.json` entry points at a hook that doesn't exist yet, because
  the hook *is* committed.
- Most of the reason `.spine/core-pin.json` needs a `strict` mode at all —
  the scenario it protects against (someone's hooks silently enforcing
  something a colleague's don't) can't happen for hooks specifically once
  they're committed content. The pin would still matter for skills/agents/
  rules staying reasonably in sync, but the stakes there are lower, so
  `warn`-only might become sufficient.

## What this costs

- **Hooks stop auto-updating with `git pull spine`.** Today, a hook fix
  (like the `jq` bug we patched) reaches every installed project the
  moment each engineer pulls their own spine clone. Under this proposal, it
  only reaches a project once someone deliberately re-syncs and commits the
  new hook content there — real update friction, project by project,
  exactly the tradeoff vendoring always carries.
- **Every hook change becomes a diff in every installed project**, not just
  in spine's own repo — more PR noise on `/update`, and a real merge-
  conflict possibility if a project ever hand-edited a hook (which nothing
  today prevents, though nothing encourages it either).
- **Migration path for already-installed projects** isn't free: existing
  projects have `.claude/hooks` gitignored and symlinked. Moving to
  committed-for-real means, for every installed project. `setup` (or a new
  one-time migration script) would need to: `git rm --cached` nothing (it's
  already untracked) — actually the opposite direction from the earlier
  Extension C migration — resolve the symlink to its real target, copy the
  real file content in, remove it from `.gitignore`, and commit it. Small
  per-project, but it's a real step every already-installed project needs,
  not just new ones going forward.
- **`core/skills/update/SKILL.md`** would need a new job: instead of (or in
  addition to) re-syncing symlinks, diff and offer to apply the new hook
  content, closer to how a package manager offers a lockfile update than
  today's "symlinks just resolve to whatever's newest automatically."

## Alternatives considered

- **Commit everything (hooks, skills, agents, rules) for real.** Rejected
  in this note for the same reason the current split targets only hooks:
  it kills the "update once, every project benefits immediately" property
  for the content that most benefits from it (skills/agents/rules churn
  more often and more harmlessly than the three core hooks). Fully covered
  under "and do you think the skills for spine should instead be in the
  project" earlier in this conversation.
- **Leave it as-is, just keep fixing skew-detection bugs as they surface.**
  The status quo — viable, and this session shows it's tractable (the two
  bugs found today were each small, real fixes). The honest case against
  it is that the *category* of bug (hooks silently absent or mismatched)
  will keep producing new instances as long as the mechanism that makes it
  possible still exists, versus fixing the shape once.

## Open questions a real design pass would need to answer

- Does `dep-gate`/`path-escalate`/`phase-gate`'s shared helper
  (`_workspace-route`, `_bash-write-targets`) get committed too, or does
  committing "the three hooks" implicitly mean committing everything they
  `source`? (Almost certainly yes — a hook that sources a still-symlinked
  helper just moves the fresh-clone problem one file over.)
- What happens to a project that's hand-edited a hook after committing it
  for real (something the symlink model made structurally impossible)?
  `/update`'s diff-and-offer flow needs a real answer here, not silence.
- Does the pin (`.spine/core-pin.json`) still need `strict` mode at all
  once hooks can't silently mismatch, or does it become purely a skills/
  agents/rules staleness signal at that point — and if so, should its
  messaging change to say that explicitly?

## Recommendation

Don't build this now. It's a real simplification with a real, honestly
one-sided cost (per-project update friction on exactly the content that
benefits most today from *not* having that friction), and it touches the
install/update contract broadly enough (`setup`, `bootstrap`/`adopt`'s wire-
the-install step, `/update`, every already-installed project's migration)
that it deserves its own scoped design pass, not a bolt-on alongside
smaller fixes. Revisit if the symlink+pin model produces another real
incident of this shape, or when there's room for a dedicated design task.
