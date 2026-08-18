# <project name>

This project runs on the spine — an AI-development workflow with
deterministic gates. Full docs: `<spine checkout path>/README.md`.

**Lost? Run `/spine`** — it tells you where you are and what to do next.

## The task system

Non-trivial work goes through `/task <description>`: classify → research →
plan (you approve it) → implement → verify → ship. Working from a ticket?
`/intake <KEY>` is the front door — it sizes the change and routes it. A trivial one-off edit
is fine to make directly — but if a hook halts you mid-edit, that's the
spine telling you it stopped being trivial; stop and run `/task` instead of
forcing the edit through some other way.

## Protected paths

`.spine/protected-paths.conf` lists this project's high-blast-radius paths
(auth, payments, migrations, dependency manifests, and whatever else this
project's own calibration named). Editing one outside a confirmed Class 2
task halts the edit — that's `path-escalate` working as intended, not a bug
to route around. A `#manifest`-tagged path (dependency files) instead forces
an explicit human decision on every edit and install command, no exceptions.

## The floor

Nothing ships without the floor passing — `/task` runs it automatically at
the verify phase against this project's `.spine/adapters/` (typecheck,
lint, test, and more depending on class). `.spine/capabilities.json`
records what's genuinely implemented here vs. unavailable for this stack,
each with a specific reason — a capability never silently skips.

## Migrations

Anything under a `migrations/`/`migration/` path follows expand/contract —
see `.claude/rules/migrations.md` (loads automatically when such a file is
read).

## Editing this file

Only `/ratchet` extends this file, and only by deleting an equal amount of
prose it supersedes with a deterministic check. Don't hand-add a rule here
that could be a test, an adapter check, or a `protected-paths.conf` entry
instead — this file is always loaded, so every line here taxes every turn,
forever.
