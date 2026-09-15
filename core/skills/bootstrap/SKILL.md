---
name: bootstrap
description: Install the spine into a brand-new, empty (or near-empty) project — charter first, then calibration, adapters, and the first /task. Run from a session in the spine repo itself.
disable-model-invocation: true
argument-hint: --project <path-to-new-project>
---

You are running `/bootstrap` against `--project <path>` from `$ARGUMENTS`
(default: current directory if the project already looks spine-shaped, but
for a genuinely new project always expect an explicit path — there's nothing
to detect yet). This is a first-class entry path, not a degraded one:
greenfield is where duplication and missing invariants get
laid down, so nothing below is skipped because the repo is empty.

If `<project>/.spine/capabilities.json` already exists, stop and say so —
this project is already installed; `/adopt` is what re-runs calibration on
an existing install, `/bootstrap` is one-time.

Scripts and templates below are referenced as
`${CLAUDE_SKILL_DIR}/../../scripts/<name>` /
`${CLAUDE_SKILL_DIR}/../../templates/<name>` — hand that to the shell
verbatim, `../../` included. Do **not** lexically collapse it to
`.claude/`; `.claude/skills/bootstrap` is a symlink into the spine core
checkout, and collapsing the text yields a nonexistent `.claude/scripts/...`
path.

## 1. Charter first — before any code exists

This is the first act, in this order, precisely because greenfield's
advantage is declaring invariants before code can contradict them. Interview
the engineer directly (don't infer — there's no code yet to infer from):
what the product is, its non-negotiables, hard constraints, what's
explicitly out of scope, and any build-order/sequencing rules ("X must
exist before Y ships") the engineer wants locked in now — the template's
own fourth bucket, `## Sequencing constraints`, exists precisely so a real
process rule doesn't get forced into "non-negotiable" or dropped for lack
of a place to put it. Write `<project>/docs/charter.md` from
`${CLAUDE_SKILL_DIR}/../../templates/charter.md`, filled in for real, marked
`DRAFT` per the template's own convention, `Source documents:` left `none`
(a greenfield interview has no source file to cite — the whole point of
asking directly). The engineer edits and confirms it before real work
starts, but do not block the rest of this skill on that — charters get
amended, they're not a gate on installation.

## 2. Layer 1 calibration (user-level, once per engineer, ever)

Check `~/.spine/user-config.json`. If present: display it, ask for a quick
confirm-or-override, don't re-interview from scratch. If absent: ask the
three Layer 1 questions — ceremony (one or two adversaries
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

## 3.5 Layer 2.5 — team strictness profile

Ask which strictness profile this repo's team runs — the default for how much
ceremony and how many human stops a task gets — and write
`<project>/.spine/profile.json` from
`${CLAUDE_SKILL_DIR}/../../templates/profile.json`, adjusting its fields to the
chosen preset (`core/ADAPTER-CONTRACT.md §7`'s table): **prototype** (one
adversary, `auto` allowed, no smoke, looser Class 0), **standard** (the
template's own values — two adversaries, `auto` allowed, smoke on), or
**regulated** (two adversaries, `autonomy_ceiling: checkpointed` so `auto` is
never offered, stricter Class 0). Default to **standard** unless the team says
otherwise. Then validate it:

```
${CLAUDE_SKILL_DIR}/../../scripts/profile-check --project <project>
```

It must pass — a profile tunes ceremony but can never disable the floor, the
protected-path hook, or the autonomy cap (§7, enforced fail-closed). This one
file is what lets several teams share one spine at different defaults; it's
ordinary committed project state, edited later like anything else.

## 4. Layer 3 calibration and adapter generation

Ask stack and commands (language, framework, test runner, typechecker,
linter, package manager — exact invocations) and runtime shape (what will
run — service, app, CLI — and a database/migration tool if any; if there is
genuinely nothing to run yet, note that honestly — a brand-new project
often doesn't, and capabilities can move from `not-applicable` to
`implemented` in a later `/adopt`-style recalibration once something exists).
These answers are consumed **only** here, in adapter generation — nothing
outside `.spine/adapters/` may ever read them, and nothing in `spine/` may
name a language, framework, or tool.

**Ask specifically, as its own question, not folded into "runtime
shape": does this project render a UI a person looks at — a
browser-rendered web app, or a native mobile/desktop app driven through a
simulator or emulator?** "App" alone doesn't distinguish a UI-rendering
app (browser or native) from a CLI or a headless service, and that
distinction is exactly what `ui-render` (`core/ADAPTER-CONTRACT.md §3.3`)
needs. If yes: ask for the glob(s) identifying view/component files and
write them to `<project>/.spine/ui-paths.conf` (one glob per line, `#`
comments — same format `.spine/protected-paths.conf` already uses), and
ask how the `ui-render` adapter will reach a real rendered state — for a
browser app, the dev-server start command + port; for a native
mobile/desktop app, the simulator/emulator boot command, the built app's
bundle/package id, and however it's driven to the relevant screen (a deep
link, or scripted login/navigation). If no: leave `.spine/ui-paths.conf`
absent (mirrors `.spine/install-command-patterns.conf`'s own "absent means
no-op" rule) and mark `ui-render` `not-applicable` in
`.spine/capabilities.json` with reason "no UI surface to render in this
project's runtime shape."

**If yes, also ask whether a UI handoff bundle (tokens, component
library, per-screen specs, screenshots — `docs/ui/`,
`core/templates/ui-handoff.md`) exists or is planned.** If not yet:
mention **`/prompts ui-handoff`** — the staged prompts for producing
one from Claude Design (or a comparable tool) — rather than letting the
engineer assume none exists. This doesn't block anything here: `ui-render`
and `ui-conformance` are independent capabilities, and a project can
add the bundle later, any time before it matters (the next `/adopt`-style
recalibration will pick it up).

For each of the 19 capabilities in `core/ADAPTER-CONTRACT.md §1`: if a real
invocation exists for the confirmed stack, write
`<project>/.spine/adapters/<name>` as a real, executable script — exit
0/non-zero, one line on success, full diagnostics on failure, plus a working
`--self-test pass`/`--self-test fail` pair, all per
`core/ADAPTER-CONTRACT.md §2–4`. If no viable tool/approach exists for this
stack, mark it `unavailable` in `.spine/capabilities.json` with the specific
reason. If the capability doesn't apply to this project's shape at all
(e.g. `smoke-*` with nothing to run yet), mark `not-applicable`, also with a
specific reason — never leave a capability unmentioned.

**Three of the eighteen are workflow adapters, not floor gates** (§3.4/§3.5/§3.6)
— generate them from the team's own tools, no separate interview question for
any of them. `ticket-fetch`: wrap the tracker the engineers actually use — its
own CLI or REST API; mark `not-applicable`, reason "no ticket source", if
tickets are always pasted by hand. `open-pr`: wrap the project's PR-host
tooling (its CLI or API); mark `not-applicable`, reason "no PR host", if PRs
are opened by hand. If a `ticket-fetch` adapter is written, also write
`<project>/.spine/ticket-pattern.conf` — one extended-regex line matching this
tracker's key shape (e.g. `[A-Z][A-Z0-9]+-[0-9]+` for `ABC-1234`), which
task startup reads to derive a ticket key from the branch name; absent,
it falls back to that same default. Also ask what branch this team's own
ticket branches actually look like (Extension F, `core/skills/task/SKILL.md`
§1's "Ticket branch"), and write `<project>/.spine/branch-naming.conf` — one
line, a template built from `{ticket}`, `{slug}`, `{type}` (`feature`|`fix`),
and `{user}` tokens, e.g. `{type}/{ticket}-{slug}`. Default if the engineer
has no particular convention: `{ticket}-{slug}`. **The template must contain
`{ticket}`** — task startup's "did the human already branch by hand"
detection depends on the ticket key literally appearing in the branch name;
refuse (ask again) a proposed template that omits it. `worktree-prep`: the
package manager and
gitignored-deps shape needed to write it are already known from the earlier
stack questions — symlink/reuse the relevant dir(s) from the source checkout
into a fresh worktree; mark `not-applicable`, reason "nothing to provision",
if this stack has no gitignored dependencies or no runnable test suite.

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
package-manager name itself (the stack-independence rule applies to hooks
too). If this project's stack has no recognizable
install-command shape, leave the file absent — `dep-gate`'s Bash check
no-ops without it, and its Edit/Write manifest-tag check is unaffected.

## 5. Wire the install

Write `<project>/.claude/settings.json` (committed, team-shared) wiring the
three hooks by `${CLAUDE_PROJECT_DIR}/.claude/hook-guard <name>` — **not**
`.claude/hooks/<name>` directly (Extension C Phase C: a PreToolUse command
pointing at a path that doesn't exist yet is silently *skipped*, not
blocked, by Claude Code's own hook runner — confirmed empirically, see
`core/templates/hook-guard`'s own header — so every committed
`settings.json` must route through the guard, which does exist from the
first commit onward, or a fresh clone before its first `setup` run is
silently unenforced). Matcher `Edit|Write|Bash` for all three (`phase-gate`,
`path-escalate`, `dep-gate` each resolve Bash file-mutation targets
themselves; see each hook's own header comment in `core/hooks/` and
`core/hooks/_bash-write-targets`). This file is portable as written —
`${CLAUDE_PROJECT_DIR}`-relative, no absolute path in it.

Then, instead of hand-symlinking (the pre-Extension-C mechanism, which
committed machine-absolute symlink targets and broke for every other
engineer's checkout):

```
${CLAUDE_SKILL_DIR}/../../scripts/setup --project <project>
```

This creates every `.claude/skills/<name>`, `.claude/agents/<name>.md`,
`.claude/rules/<name>.md`, `.claude/hooks` symlink (still one per skill/
agent/rule, still pointing at this checkout — the *set* of names doesn't
change, only how the pointer gets there and whether it's committed), adds
them to `<project>/.gitignore` (generated locally on every machine from
here on, never committed), merges the machine-local Bash-allow pattern into
`<project>/.claude/settings.local.json`, and initializes
`<project>/.spine/core-pin.json` at this checkout's current `HEAD` sha,
mode `warn` (Extension C §2.1 — a maintainer bumps this deliberately after
testing a newer core; see the README's "Staying installed" section).

Write `<project>/CLAUDE.md` from
`${CLAUDE_SKILL_DIR}/../../templates/CLAUDE.md` verbatim — including its
`<!-- spine:begin -->`/`<!-- spine:end -->` markers, even on a brand-new
project with no prior content — filled with this project's real
task-system pointer and floor invocation. Never strip the markers just
because the file is starting from empty: the moment this engineer adds
their own content below `spine:end`, it must never be blocked by the
60-line cap or need a later migration into markers. **Count the lines
inside the markers — hard cap 60.** If your fill-in pushed it over, cut,
don't shrink the font: this file is always-loaded, every line inside the
block taxes every future turn.

Write `<project>/docs/map.md` from
`${CLAUDE_SKILL_DIR}/../../templates/map.md` — leave every section
near-empty, stamp it with current `HEAD` (or a fresh initial commit if the
project has none yet) and the current timestamp. An empty map with a fresh
stamp is correct here; it grows through `/remap` as the codebase does.

`mkdir -p <project>/docs/decisions <project>/work` and touch `.gitkeep` in
`docs/decisions`. Create `<project>/work/.gitignore` with this exact content
(committed to the repo so every team member gets it):

```
# Ignore task telemetry/tracking files — keep narrative docs (research.md,
# plan.md, deviations.md, verify.md, briefing.md, notes.md) which are
# committed by registry-sync and provide context for future sessions.
*/state
*/class
*/owner
*/autonomy
*/flags.json
*/claims.json
*/approval.json
*/ticket
*/milestone
*/artifacts/
```

This lets `registry-sync` commit only the human-readable per-task docs while
leaving internal tracking files off the PR diff entirely.

## 6. Commit the install and hand off

Commit the symlinks, `.claude/settings.json`, `docs/charter.md`,
`docs/map.md`, `docs/decisions/.gitkeep`, `work/.gitignore`,
`.spine/capabilities.json`, `.spine/protected-paths.conf`,
`.spine/install-command-patterns.conf` (if written),
`.spine/ui-paths.conf` (if written), `.spine/profile.json`,
`.spine/ticket-pattern.conf` (if written), `.spine/branch-naming.conf` (if
written), and
`.spine/adapters/` as one setup commit — this is the one commit any
install mechanism requires; everything after this is `git pull` inside
`spine/` with zero further commits in `<project>`.

**Tell the engineer to start a fresh session in `<project>` before doing
anything else.** Hook and skill wiring does not hot-reload mid-session —
the session that ran this install will not see the newly-symlinked skills
or armed hooks. From that fresh session, `/task` is the first real command
to run.

**If the engineer has a product spec, design handoff, or build plan** (a
document describing what the product does and the intended build sequence),
tell them to save it as `docs/product-spec.md` — not to hand-extract a
milestone list from it themselves. Two real mechanisms exist to turn it
into decisions and milestones the *right* way, and `/design` (the next
real setup command after this one) auto-detects `docs/product-spec.md` and
offers to use it: (1) `/design` reads it alongside the charter so the six
foundational categories are actually grounded in what the document says,
not guessed at afterward; (2) once decisions exist, if the remaining shape
is still genuinely foggy, `/wayfinder` charts it into a map of decision
tickets and writes `docs/vision.md`'s milestone list once that map clears
— `/roadmap` sequences it from there the same way it would a hand-written
one. **Do not extract `docs/vision.md` yourself at this point** — no
decisions exist yet, so there's nothing to ground a milestone breakdown
in; a milestone list assembled before `/design` has run is exactly the
failure mode `/wayfinder` exists to prevent (a large document read once,
cold, and committed to in one pass, with no chance for the six
architecture categories or a decision map to catch what a milestone-first
read misses). If the engineer has no such document yet, they can always
add `docs/product-spec.md` later — `/design` checks for it every run, not
just the first — and **`/prompts product-spec`** prints a ready-to-paste
interview prompt for drafting one in a separate chat session, in exactly
the structure `/design` expects. Mention it here plainly; don't assume
they already know it exists. Skip this note entirely for Jira-driven work
where the ticket queue is the plan.
