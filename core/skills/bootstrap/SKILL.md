---
name: bootstrap
description: Install the spine into a brand-new, empty (or near-empty) project — charter first, then calibration, adapters, and the first /task. Run from a session in the spine repo itself.
disable-model-invocation: true
argument-hint: --project <path-to-new-project>
---

You are running `/bootstrap` against `--project <path>` from `$ARGUMENTS`
(default: current directory if the project already looks spine-shaped, but
for a genuinely new project always expect an explicit path — there's nothing
to detect yet). This is a first-class entry path (build prompt §2.2), not a
degraded one: greenfield is where duplication and missing invariants get
laid down, so nothing below is skipped because the repo is empty.

If `<project>/.spine/capabilities.json` already exists, stop and say so —
this project is already installed; `/adopt` is what re-runs calibration on
an existing install, `/bootstrap` is one-time.

## 1. Charter first — before any code exists

This is the first act, in this order, precisely because greenfield's
advantage is declaring invariants before code can contradict them. Interview
the engineer directly (don't infer — there's no code yet to infer from):
what the product is, its non-negotiables, hard constraints, what's
explicitly out of scope. Write `<project>/docs/charter.md` from
`${CLAUDE_SKILL_DIR}/../../templates/charter.md`, filled in for real, marked
`DRAFT` per the template's own convention. The engineer edits and confirms
it before real work starts, but do not block the rest of this skill on
that — charters get amended, they're not a gate on installation.

## 2. Layer 1 calibration (user-level, once per engineer, ever)

Check `~/.spine/user-config.json`. If present: display it, ask for a quick
confirm-or-override, don't re-interview from scratch. If absent: ask the
three Layer 1 questions (build prompt §0) — ceremony (one or two adversaries
on Class 1; smoke in the floor when its runtime fits the budget), budget
sensitivity (default: no cap), work mode (default: solo, single-stream) —
and write the file. This file is never project-specific; once it exists,
every future `/bootstrap` or `/adopt` on this machine reads it, never
re-asks.

## 3. Layer 2 calibration (repo-level)

Protected paths on a new project are declared **in advance of the code that
will occupy them** — ask the engineer what's planned (auth, payments, PII,
public API surface, migrations directory name) even though none of it
exists on disk yet, and write `<project>/.spine/protected-paths.conf` from
those answers (format: one glob per line, `#migration`/`#manifest` tags —
see `core/hooks/path-escalate` and `core/hooks/dep-gate` for exactly how
each tag is consumed). Confirm risk posture (default: any protected-path
touch forces Class 2) and CI integration (default: mirror locally; wiring
an actual CI pipeline is an addendum the engineer does themselves, not a
dependency of this install).

## 4. Layer 3 calibration and adapter generation

Ask stack and commands (language, framework, test runner, typechecker,
linter, package manager — exact invocations) and runtime shape (what will
run — service, app, CLI — and a database/migration tool if any; if there is
genuinely nothing to run yet, note that honestly — a brand-new project
often doesn't, and capabilities can move from `not-applicable` to
`implemented` in a later `/adopt`-style recalibration once something exists).
These answers are consumed **only** here, in adapter generation — nothing
outside `.spine/adapters/` may ever read them (build prompt §2.5), and
nothing in `spine/` may name a language, framework, or tool.

For each of the 13 capabilities in `core/ADAPTER-CONTRACT.md §1`: if a real
invocation exists for the confirmed stack, write
`<project>/.spine/adapters/<name>` as a real, executable script — exit
0/non-zero, one line on success, full diagnostics on failure, plus a working
`--self-test pass`/`--self-test fail` pair, all per
`core/ADAPTER-CONTRACT.md §2–4`. If no viable tool/approach exists for this
stack, mark it `unavailable` in `.spine/capabilities.json` with the specific
reason. If the capability doesn't apply to this project's shape at all
(e.g. `smoke-*` with nothing to run yet), mark `not-applicable`, also with a
specific reason — never leave a capability unmentioned.

Then:

```
${CLAUDE_SKILL_DIR}/../../scripts/adapter-conformance --all --project <project>
```

**Do not mark a capability `implemented` in `capabilities.json` unless this
passes for it.** An adapter that fails its own known-fail case is not
conformant regardless of how correct its known-pass case looks.

Also write `<project>/.spine/install-command-patterns.conf`: one
extended-regex fragment per line (grep -E syntax) recognizing this
project's own dependency-install commands — derived from the confirmed
package manager(s), same answers as adapter generation above. This is what
`core/hooks/dep-gate` reads for its Bash-command check; it never hardcodes a
package-manager name itself (build prompt §3's stack-independence rule
applies to hooks too). If this project's stack has no recognizable
install-command shape, leave the file absent — `dep-gate`'s Bash check
no-ops without it, and its Edit/Write manifest-tag check is unaffected.

## 5. Wire the install

Create these symlinks inside `<project>` (absolute paths to this spine
checkout — confirmed mechanism, `phase-A-handoff.md` §4):

```
.claude/skills/<name>  -> <spine>/core/skills/<name>   (one per skill)
.claude/agents/<name>.md -> <spine>/core/agents/<name>.md  (one per agent)
.claude/rules/<name>.md  -> <spine>/core/rules/<name>.md   (one per rule)
.claude/hooks -> <spine>/core/hooks   (whole-directory symlink)
```

Write `<project>/.claude/settings.json` (committed, team-shared) wiring the
three hooks by `${CLAUDE_PROJECT_DIR}/.claude/hooks/<name>` path — matchers
`Edit|Write` for `phase-gate` and `path-escalate`, `Edit|Write|Bash` for
`dep-gate` (see each hook's own header comment in `core/hooks/`).

Write `<project>/CLAUDE.md` from
`${CLAUDE_SKILL_DIR}/../../templates/CLAUDE.md`, filled with this project's
real task-system pointer and floor invocation. **Count its lines — hard cap
60.** If your fill-in pushed it over, cut, don't shrink the font: this file
is always-loaded, every line taxes every future turn.

Write `<project>/docs/map.md` from
`${CLAUDE_SKILL_DIR}/../../templates/map.md` — leave every section
near-empty, stamp it with current `HEAD` (or a fresh initial commit if the
project has none yet) and the current timestamp. An empty map with a fresh
stamp is correct here; it grows through `/remap` as the codebase does.

`mkdir -p <project>/docs/decisions <project>/work` and touch `.gitkeep` in
each.

## 6. Commit the install and hand off

Commit the symlinks, `.claude/settings.json`, `docs/charter.md`,
`docs/map.md`, `docs/decisions/.gitkeep`, `work/.gitkeep`,
`.spine/capabilities.json`, `.spine/protected-paths.conf`,
`.spine/install-command-patterns.conf` (if written), and
`.spine/adapters/` as one setup commit — this is the one commit any install
mechanism requires (build prompt §3); everything after this is `git pull`
inside `spine/` with zero further commits in `<project>`.

**Tell the engineer to start a fresh session in `<project>` before doing
anything else.** Hook and skill wiring does not hot-reload mid-session
(`phase-A-handoff.md` §2.1) — the session that ran this install will not see
the newly-symlinked skills or armed hooks. From that fresh session, `/task`
is the first real command to run.
