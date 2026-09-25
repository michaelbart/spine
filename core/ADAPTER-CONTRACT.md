# Adapter contract

Authoritative for every file in `core/scripts/`, every adapter in a project's
`.spine/adapters/`, and `core/scripts/adapter-conformance`. Nothing here names
a language, framework, or tool — this document, like the core, is stack-blind.

## 1. What a capability is

A capability is an executable at `.spine/adapters/<name>` in the installed
project. The core never calls a tool directly; it calls a capability name.
Twenty capabilities exist:

`typecheck`, `lint`, `test`, `test-changed`, `secret-scan`, `dep-diff`,
`clone-scan`, `callers`, `mutate`, `smoke-seed`, `smoke-run`, `smoke-golden`
(§3.8), `migrate-rehearse` (§3.7), `contract-check` (Extension B — §3.2),
`ui-render` (§3.3), `ui-conformance` (§3.9), `ui-capture` (§3.10), `ticket-fetch` (intake —
§3.4), `open-pr` (ship — §3.5), `worktree-prep` (falsifier — §3.6).

`contract-check` only ever exists in a repo that is a workspace member and
party (producer or consumer) to at least one declared contract
(`workspace.json`'s `contracts[]`, `core/skills/workspace/SKILL.md`) — a
plain single-repo project marks it `not-applicable` with reason "not a
workspace member," the same disclosed-degradation shape every other
inapplicable capability already uses. Unlike the other thirteen (`ui-render`
shares this trait too, see below), `contract-check` is never invoked by
`core/scripts/floor`'s own dispatch loop (floor's fixed capability
sequence is unchanged by Extension B, preserving single-repo behavior
exactly) — `core/skills/verify/SKILL.md`'s aggregation step invokes it
directly, once per touched contract, only for repos
`core/scripts/contract-touch` puts in a task's blast radius. See §3.2.

`ui-render` only ever exists in a project whose runtime shape renders a UI
a person looks at — a browser-rendered web app, or a native mobile/desktop
app driven through a simulator or emulator (`core/skills/bootstrap/
SKILL.md` / `core/skills/adopt/SKILL.md`'s Layer 3 calibration question) —
a project with no such surface (a CLI, a library, a pure API with no
rendered surface) marks it `not-applicable` with reason "no UI surface to
render in this project's runtime shape." Like `contract-check`, `ui-render` is never invoked by
`floor`'s dispatch loop — `core/skills/verify/SKILL.md`'s own orchestration
invokes it directly, and only when `core/scripts/ui-touch` reports the
task's diff actually touched a UI-file-shape path (`.spine/ui-paths.conf`).
See §3.3.

`ui-conformance` only ever exists in a project with a UI handoff
bundle (`docs/ui/handoff.md`, `core/templates/ui-handoff.md`) — a
project with no such bundle marks it `not-applicable` with reason "no
UI handoff bundle (docs/ui/) in this project." Unlike `ui-render`'s
own conditional question, this one needs no dedicated interview question
at bootstrap/adopt time — bundle presence is a plain file check, not a
judgment call the way "does this project render a UI a person looks at"
is. Like
`ui-render`, it is never invoked by `floor`'s dispatch loop —
`core/skills/verify/SKILL.md`'s own orchestration invokes it directly,
gated by the same `core/scripts/ui-touch` result §3.3 already uses. See
§3.9.

`ui-capture` only ever exists in a project whose `ui-render` is implemented
and whose handoff bundle carries reference screenshots. Like `ui-conformance`
it is never invoked by `floor`'s dispatch loop — `/verify` step 1e runs it,
gated by `ui-touch` and its own class opt-in, and hands its output to the
`ui-fidelity` reviewer. See §3.10.

`ticket-fetch` only ever exists in a project whose engineers work from an issue
tracker (`/intake`, `core/skills/intake/SKILL.md`) — a project with no tracker,
or one where tickets are always pasted by hand, marks it `not-applicable` with
reason "no ticket source" and `/intake` falls back to a manual paste. Like
`contract-check` and `ui-render`, it is never invoked by `floor`'s dispatch loop
— only `/intake` calls it, once, to fetch the ticket it was handed. Unlike every
other capability it is a *data source*, not a pass/fail gate: exit 0 means the
ticket was fetched (JSON on stdout), non-zero means it couldn't be (→ manual
paste). See §3.4.

`open-pr` only ever exists in a project whose engineers ship through pull
requests on a host the project's own PR tooling can reach (`core/skills/ship/SKILL.md` §5a) —
a project with no PR host, or one where PRs are always opened by hand, marks it
`not-applicable` and `/ship` falls back to committing locally and leaving the PR
to the human. Like `ticket-fetch`, it is invoked only by a skill (`/ship`), never
by `floor`, and it is an *action* adapter, not a gate: exit 0 means a **draft** PR
was opened (its URL on stdout), non-zero means it couldn't be. It **never** opens
a ready-to-merge PR and never merges — the human marks ready and merges. See §3.5.

`worktree-prep` only ever exists in a project with gitignored dependencies
(`node_modules`, `.venv`, `target/`, or equivalent) and a runnable test
suite — a project with neither marks it `not-applicable` with reason
"nothing to provision" (nothing a fresh worktree would be missing). It is
**never invoked by `floor`** — the falsifier alone runs it, once, as the
first bounded step of its stub-out probe (`core/agents/falsifier.md`
mandate (b)), never `security` (read-only, no worktree isolation). Like
`open-pr`, it is an *action* adapter, not a gate: exit 0 means the
project's toolchain is now resolvable in the current (isolated) worktree,
non-zero means it couldn't be. See §3.6.

`.spine/capabilities.json` marks each `implemented`, `unavailable`, or
`not-applicable`, each with a `reason`. Only capabilities marked `implemented`
are ever invoked by `floor` or any skill. An absent or degraded capability is
never silently skipped — its gate is recorded as degraded, explicitly, in
`verify.md` and the delta briefing.

## 2. Exit code and output discipline

Every adapter, in every mode, obeys:

- **Exit 0 means pass. Any non-zero exit means fail.**
- **On pass: stdout is exactly one line.** No exceptions. This is what keeps
  a floor run's own output small enough to stay in context.
- **On fail: stdout/stderr carries full diagnostics** — everything a human or
  an adversary subagent would need to see the failure without re-running the
  command themselves.
- Adapters never prompt. Never read from a TTY. Never mutate files outside
  what the capability inherently requires (e.g. a formatter check must not
  reformat in place; that's a different capability if it's ever wanted).

### 2.1 Shell strictness and bash portability

"Exit 0 means pass" (§2 above) is only as trustworthy as the adapter's own
ability to *notice* it failed. `floor` (`run_simple`/`run_with_stdin`/
`run_scoped`/`run_artifact`, `core/scripts/floor`) never inspects an
adapter's stderr to decide pass/fail — it trusts the exit code alone, by
design (§2). A bash adapter that hits a real error mid-script but doesn't
actually terminate keeps running, can still reach its own `exit 0` (or fall
off the end of the script, which is the same thing), and `floor` records a
clean pass over a crashed, garbage run. This is not hypothetical: a
project's own `typecheck` adapter used `local -A` (bash 4+ associative
arrays) under `set -uo pipefail` (no `-e`); `/bin/bash` on an unmodified
macOS is bash 3.2 (Apple has not shipped a newer bash since the GPLv3
switch — this is the default on every contributor's Mac, not a misconfigured
one), so `local -A` failed with "invalid option," an unbound-variable
reference later in the same function printed to stderr, and the script
kept going and exited 0. `floor` recorded PASS.

Every bash adapter — and `core/scripts/*` itself — therefore must:

- **Start with `set -euo pipefail`, not a weaker subset.** `-u` alone
  (unbound-variable errors print but don't stop the script) is not
  sufficient; `-e` is what turns a real error into the non-zero exit
  `floor` actually gates on. An adapter with a legitimate reason for a
  specific command to fail without aborting handles that command
  explicitly (`cmd || true`, an `if` check, etc.) — `set -e` stays on for
  everything else.
- **Not assume bash 4+ features** — associative arrays (`declare -A`/
  `local -A`), `readarray`/`mapfile`, `&>>`, and similar — **unless the
  adapter itself checks `$BASH_VERSINFO`** and either degrades to a
  bash-3.2-compatible path or fails loudly with a clear "needs bash 4+"
  diagnostic (never silently, per §2's diagnostics rule). Adapters are
  meant to be portable across contributors' own machines, not just
  whatever bash CI happens to run — macOS's default `/bin/bash` is the
  floor to write against, not the exception to special-case.

`core/scripts/adapter-conformance` lints every bash-shebang adapter
(`#!/bin/bash` or `#!/usr/bin/env bash`) for both of these mechanically,
before running its self-test fixtures (§4) — a missing `set -euo pipefail`
or an unguarded bash-4-only construct fails conformance outright, the same
fail-closed shape every other conformance check uses. This is a static,
best-effort lint (it reads the adapter's source; it does not prove every
code path is unreachable-without-`-e`-safe), not a substitute for the rule
above — the rule is what a human or AI adapter author must follow either
way.

### 2.2 Shared environment state (stack lifecycle)

Nothing in this contract coordinates state between adapters. `floor`'s
dispatch loop invokes capabilities in a fixed sequence (§3.7, §3.8), but
that sequence is not a promise about what any later adapter finds when it
starts — a human re-running one capability directly, a resumed partial
floor run after a fix, or a skill invoking a single capability out of
band, all bypass it. Two rules follow, for any capability whose
correctness depends on a running local stack or service:

- **If a capability's own work requires the stack to be up, it must bring
  the stack up itself when it finds it down** — self-heal, not fail with
  a diagnostic asking whether it's running. `smoke-run`, `smoke-seed`, and
  `smoke-golden` (§3.8) are the current instances of this; any future
  capability with the same dependency inherits the same requirement.
  Treating "stack not running" as an environment precondition someone
  else must satisfy pushes a cross-adapter coordination problem onto
  whatever happens to run first — which is exactly the shape of bug this
  rule exists to close: a capability earlier in `floor`'s sequence tore
  the stack down as an unrelated side effect of its own cleanup, and the
  next one in sequence had no way to know that mattered, because sequence
  order was never a contract either adapter could rely on.
- **If a capability starts the stack itself as a means to its own end**
  (rather than because the capability's whole point is to run against
  it), it must restore what it found, not what its own internals happen
  to need at exit — leave the stack running if it found it already
  running, stop it only if it started it. `migrate-rehearse` (§3.7) is
  the current instance: it needs the stack up to rehearse a migration,
  but "the stack was already up when I started" and "I brought the stack
  up to do my job" are different situations, and only the second one
  licenses tearing it back down on the way out.

Both rules exist because `floor` trusts each adapter's exit code alone
(§2) and never inspects what state an adapter leaves behind — the same
trust boundary §2.1 describes for shell strictness applies here to
environment state: an adapter's own discipline is the only thing standing
between "floor: PASS" and a stack that's silently gone. Confirmed real,
not hypothetical: a project's `migrate-rehearse` stopped the local stack
unconditionally on its way out (its own cleanup discipline, correct in
isolation), and the `smoke-seed` that ran immediately after it in the
same `floor` invocation had no restart fallback — it resolved an anon key
via a status check and failed with "could not resolve the local stack's
anon key... is it running?" on the first Class 2 task that touched a
migration, regardless of whether the migration itself was correct.

## 3. Invocation convention

All adapters run with CWD set to the project root (the directory containing
`.spine/`). Beyond that, the convention is deliberately small:

| Input needed | Convention |
|---|---|
| Nothing (operates on the whole working tree) | No args, no stdin. Applies to `test`, `mutate`, `smoke-seed`, `smoke-run`, `smoke-golden`, `migrate-rehearse`. |
| A changed-file set | Newline-separated relative paths on **stdin**. Applies to `test-changed`, `clone-scan`, `lint`, `typecheck`. `secret-scan` accepts the same on stdin as a scope-narrowing *option*; if stdin is empty or absent it falls back to scanning the full tracked tree. |
| An output artifact path | **First positional argument.** Applies to `callers` and `dep-diff` — both additionally emit an artifact (the caller map, the dependency diff) to that path, in whatever format the adapter defines, while pass/fail semantics stay governed by exit code alone. `callers` also reads the changed-file/symbol set from stdin. |
| A comparison base for anything diffed against history (`dep-diff`, and any adapter that needs one) | Environment variable `SPINE_BASE_REF`, default `HEAD`. Never a positional argument — keeps the positional slot reserved for the output path. |

No capability takes more than one positional argument. This is the single
convention every adapter and `floor` itself must obey.

### 3.1 `lint`/`typecheck` rescoping (Phase E) and the `--full` escape hatch

Through Phase D, `lint` and `typecheck` were whole-tree, no-stdin
capabilities. Phase E moved them to the changed-file-set bucket: a task's
floor result must reflect the task's own change, not the repository's
accumulated history, per the architecture decision recorded in
`docs/tradeoffs.md` (the "rescope, don't just add `--full`" question the
build declined to resolve unilaterally in Phase D — now decided). A
diff-scoped adapter is free to still invoke its underlying tool
whole-tree internally where the tool needs full-project context for
correct results (a type checker resolving imports, for instance) — the
contract governs what *input* the adapter receives and what *scope* its
pass/fail verdict covers, not how it shells out internally. What must not
happen is a whole-tree-scoped verdict failing a task for a violation in a
file the task never touched.

`lint` and `typecheck` additionally accept a `--full` flag: no stdin is
read (any piped stdin is ignored), and the adapter runs its full,
whole-tree check exactly as it did through Phase D. This is the only
sanctioned way to invoke either capability without a changed-file set —
mirroring the `--self-test` extension mechanism in §4, a second invocation
shape layered on top of the base convention, not a replacement for it.
`floor --full` (below) is the intended caller; a per-task `/task` run must
never pass `--full`. This exists because whole-tree enforcement still has
a real home — CI, and a dedicated cleanup task that pays down pre-existing
debt once — it has simply been moved out of the per-task gate, which was
failing every task in horizon for reasons no single task's diff could fix.

**Multi-project solutions and platform-head coverage.** A solution with
more than one buildable project (a shared library plus one or more
platform-head/executable projects — a mobile app's iOS/Android heads, a
monorepo's separate deployable packages) has a second scope axis beyond
the file-set scoping above: which *projects* a `typecheck` run attempts to
build at all, not just which *files* within one project it covers. A
`typecheck` adapter that only ever compiles the library project — because
it's the one buildable without extra credentials, or simply the fastest —
can report a clean PASS while the platform-head project the diff actually
changed, and that a person would actually ship, was never compiled. This
is the same shape of blind spot §3.3 documents for `test`/`test-changed`
versus `ui-render` (a check running against a stand-in passes while the
real thing is broken): a `typecheck` adapter for a multi-project solution
must attempt to build every project in the diff's blast radius that
produces a shippable artifact, not merely whichever project is most
convenient to build offline. Where a platform-head project genuinely
cannot build in this environment — a private package feed with no
credentials configured, for instance — that is not a reason to narrow
what `typecheck` attempts; it is exactly what §2's exit-code discipline
exists to surface. The adapter's own build step fails, `typecheck` reports
FAIL with the real diagnostic (the feed-auth error, or whatever it
actually was) on stdout/stderr, and a human sees an honest, actionable
gate instead of a clean PASS that quietly covered less than it looked
like.

### 3.2 `contract-check` (Extension B) — which contract, not an output path

`contract-check` needs an input §3's table has no row for: **which
declared contract to check conformance against** — a repo can be party to
more than one. Two environment variables, following the same precedent
`SPINE_BASE_REF` already sets (an environment variable, not a second
positional argument — §3's "no capability takes more than one positional
argument" rule is unchanged):

| Variable | Carries |
|---|---|
| `SPINE_CONTRACT_NAME` | The contract's `name` from `workspace.json`'s registry. |
| `SPINE_CONTRACT_SPEC_PATH` | Absolute path to the spec file the caller already resolved (`<workspace-root>/<spec_path>`) — the adapter, run with CWD at its own repo root, has no way to find the workspace root on its own and is not expected to; the caller (`core/skills/verify/SKILL.md`'s aggregation, which already knows the workspace root) resolves it. |

No stdin, no positional argument, exit code alone governs pass/fail (§2) —
`contract-check` sits in §3's "operates on nothing" row in spirit, plus
these two required env vars. The check itself is stack-specific and
symmetric: a producer's `contract-check` verifies its real implementation
still matches the spec at `SPINE_CONTRACT_SPEC_PATH`; a consumer's verifies
its own usage still matches it (a generated client is current, a query
shape matches a schema, whatever that stack's truth is) — same capability
name, same env-var contract, opposite direction of proof, exactly as
`typecheck`/`lint` are the same capability name across every stack despite
checking entirely different things.

**Self-test**: `--self-test pass`/`--self-test fail` build their own
throwaway spec fixture inside scratch space, same as every other
capability (§4) — they do not read `SPINE_CONTRACT_SPEC_PATH` at all, since
self-test's entire point is proving the adapter's own logic without
touching anything real. An adapter that requires the real env vars to be
set even in self-test mode has misunderstood the convention.

### 3.3 `ui-render` — a real render, not a mocked one

Every other capability that touches UI code (`test`, `test-changed`) runs
against a mocked DOM or mocked network layer — real, valuable coverage,
but structurally blind to a class of bug that only shows up when the
actual view is rendered by the real runtime a person would see it in, not
a mock: for a browser-rendered app, the actual view served by the actual
dev server and rendered in an actual browser (a network call that returns
something other than what the test mocked — a dev-server SPA fallback
returning 200+HTML for an unmatched API route, for instance — a
CSS/layout failure invisible to jsdom/happy-dom, a route that 404s for
real); for a native mobile/desktop app, the actual screen running on an
actual simulator/emulator (a deep link or navigation step that silently
no-ops, a screen that never leaves a loading state against a real
backend). `ui-render` exists to catch exactly this gap, not to replace or
duplicate `test`/`test-changed`.

`ui-render` sits in §3's "operates on nothing" row: no stdin, no
positional argument, exit code alone governs pass/fail (§2). The adapter
is responsible end-to-end for standing up whatever it needs to reach a
real rendered state, and tearing down anything it started before
returning, pass or fail — for a browser-rendered app: starting the
project's own dev server (or reusing one already running) and driving a
real or headless browser to a real route; for a native mobile/desktop
app: booting a simulator/emulator (or reusing one already booted),
installing and launching the built app, and driving it to the relevant
screen (a deep link, or scripted login/navigation). Which routes/screens
to drive and how to reach a rendered state is a stack-specific detail the
adapter itself owns (e.g. reading a small project-local routes/screens
list it maintains, or the same entry points `docs/map.md`'s survey
already names) — core never enumerates routes or screens on the
adapter's behalf.

**Pass criterion, at minimum**: every route/screen the adapter drives
renders a non-empty, non-whitespace body (browser) or a non-blank,
non-crashed screen (native), and (where the adapter can determine it) the
rendered content is one of that route/screen's own known states — not a
generic framework shell, not an empty successful-request placeholder, not
a permanently-loading spinner. An adapter that only checks "the page
returned a 200" or "the app launched" has not implemented this capability
correctly; the entire reason it exists is to catch a blank/wrong render
that a 200 status or a successful launch would not.

**Eligibility, not always-on**: unlike `test`, `ui-render` is not run
unconditionally by `floor` — it is never invoked by `floor`'s dispatch
loop at all. `core/skills/verify/SKILL.md`'s own orchestration runs
`core/scripts/ui-touch` against the task's real diff first; `ui-render`
is only invoked when that reports the diff touched a UI-file-shape path
(`.spine/ui-paths.conf`, project-declared, written at `/bootstrap`/`/adopt`
time). A task whose diff never touches a view/component file never pays
this capability's cost.

**Never a pixel/perceptual diff.** Same rule §3.9 states explicitly for
`ui-conformance`: a real render, whether captured via a browser or a
simulator/emulator screenshot, is grounding material and a pass/fail
input for the criterion above — it is deliberately never diffed against a
golden image. Pixel- or perceptual-diffing is exactly the kind of gate
that's flaky across font rendering, anti-aliasing, animation timing, and
dynamic content without dedicated image-diff infrastructure this core
does not ship, and native rendering surfaces are, if anything, more prone
to that flakiness than a browser (status bars, keyboard/permission
overlays, animation frames) — a flaky floor-adjacent gate erodes trust in
every other gate next to it.

**Self-test**: `--self-test pass`/`--self-test fail` build their own
throwaway fixture inside scratch space — a throwaway page plus a
throwaway static server to serve it (browser case), or a throwaway
fixture screen in a scratch build plus a scratch simulator/emulator
target (native case) — never the project's real dev server, real routes,
or real app bundle, same isolation every other capability's self-test
already requires (§4). Both
modes must route through the *same* render-check function normal mode
uses (§4's own general rule, restated here because this is exactly the
capability the rule was written after finding broken elsewhere): the pass
fixture renders visible, non-blank text; the fail fixture renders an
empty/blank body, and the adapter's self-test must actually detect that
via its real check, not a simplified stand-in.

### 3.4 `ticket-fetch` (intake) — a data source, not a gate

`/intake` hands `ticket-fetch` a single ticket identifier and expects the ticket
back, so this adapter deliberately breaks the two §2/§3 rules that only make
sense for pass/fail gates, and no others:

- **Input**: the ticket key or URL as its **single positional argument** (e.g.
  `ABC-1234`). This is the one adapter whose positional carries an *input*, not
  an output-artifact path (§3) — it fetches data rather than producing a check
  artifact. No stdin.
- **Output on success (exit 0)**: a single JSON object on stdout — `{key, title,
  description, url}` required, `{status, comments, links}` optional — the
  clarified-brief source `/intake` reads. (The one adapter whose pass-mode stdout
  is structured data, not the one-line summary §2 requires of a gate.)
- **Output on failure (non-zero)**: diagnostics on stderr; `/intake` treats any
  non-zero exit (no such ticket, auth failure, tracker unreachable, or no adapter
  installed at all) as "couldn't fetch" and falls back to asking the human to
  paste the ticket. A fetch failure is never a hard stop — the front door still
  opens, just by hand.
- The §2 rules that still apply: never prompts, never reads a TTY, never mutates
  the tree. Credentials come from whatever environment the adapter's own
  implementation arranges (it wraps whatever tracker CLI or API the project uses), never an interactive prompt.

**Self-test** (§4): `--self-test pass` returns a well-formed fixture ticket JSON
(exit 0); `--self-test fail` returns malformed/empty output with a non-zero exit
— proving `/intake` can tell a real fetch from a failed one without a live
tracker.

### 3.5 `open-pr` (ship) — an action, not a gate

`/ship` calls `open-pr` for a task whose autonomy is `auto` or `checkpointed`
(`core/skills/task/SKILL.md`) to push the current branch and open a **draft** PR,
so the human's one remaining touchpoint is reviewing/merging it. Inputs by
environment variable (the §3.2 precedent — no positional, since there is no
output-artifact path and more than one input):

| Variable | Carries |
|---|---|
| `SPINE_PR_TITLE` | The PR title (typically the commit subject). |
| `SPINE_PR_BODY_FILE` | Path to the PR body — always `work/<task-id>/pr-description.md`, already written by `/ship` §4a. |
| `SPINE_PR_BASE` | Optional base branch; if unset the adapter uses the project's own mainline convention (e.g. `develop`). |
| `SPINE_PR_HEAD` | Optional head branch; if unset the adapter uses the current branch. |

- **Output on success (exit 0)**: the PR URL on stdout (`/ship` records it in the
  briefing). **Always a draft.** The adapter must never open a ready PR and never
  merge — that is the human's decision (proposal §6.2).
- **Output on failure (non-zero)**: diagnostics on stderr; `/ship` degrades to the
  `guided` behavior (commit already made locally — push and open by hand) and
  records the gap. A failed PR-open is never a lost commit.
- The §2 rules that apply: never prompts, never reads a TTY. Credentials come from
  whatever environment the adapter arranges (it wraps whatever push-and-PR tooling the project already uses).

**Self-test** (§4): `--self-test pass` prints a well-formed fixture PR URL and
exits 0; `--self-test fail` exits non-zero — proving `/ship` can tell a real open
from a failure without a live host. (Neither self-test contacts a real host.)

### 3.6 `worktree-prep` — an action, not a gate

The falsifier runs in an isolated git worktree (`core/agents/falsifier.md`,
`isolation: worktree`) so it can stub code and run mutated tests without
touching the implementer's real tree. But a fresh worktree lacks the
project's gitignored dependencies, so the stub-out probe's test runner
often can't resolve at all — the `auto` fire-test's finding
(`docs/tradeoffs.md`, "The `auto` fire-test happened"). `worktree-prep`
closes that gap the same way every other stack-specific action does: a
capability adapter the project owns, invoked by name only.

`worktree-prep` sits in §3's "operates on nothing" row: no stdin, no
positional argument, exit code alone governs pass/fail (§2). It runs with
CWD at the isolated worktree the falsifier is already in, and is
responsible for making that worktree's test runner resolvable — typically
by symlinking or otherwise reusing the gitignored dependency dir(s) from
the **source checkout**, discoverable stack-blind via `git worktree list`
(which, run from inside a worktree, names the main worktree it was cloned
from). Symlink/reuse is strongly preferred over a clean install: it's
near-instant, and the entire point of this capability is to *not* burn the
adversary's budget re-provisioning what the source checkout already has.
A clean install is an acceptable fallback only when reuse isn't safe (e.g.
a dependency dir that itself needs to be built per-worktree).

**Output on success (exit 0)**: one line on stdout (§2) — the toolchain is
now resolvable. **Output on failure (non-zero)**: full diagnostics on
stderr; the falsifier records the existing "stub-out probe: toolchain
unavailable in isolated worktree" tooling-gap verdict and leans on its
other mandates, exactly as it does today when no adapter exists at all. A
failed prep is never fatal to the falsifier's run — it is a disclosed
degradation, same shape as every other capability's absence (§1).

The §2 rules that apply: never prompts, never reads a TTY, never mutates
anything outside the worktree it's running in.

**Eligibility**: `not-applicable` for a stack with no gitignored
dependencies or no runnable test suite (nothing to provision);
`unavailable` if there's a real need but no viable mechanism for this
stack. Same disclosed-degradation shape every other capability uses (§1).

**Self-test** (§4): `--self-test pass` proves that a depless scratch
checkout, built inside the adapter's own temp scratch space, has its
toolchain resolvable *after* the adapter runs against it (exit 0, one
line) — never the project's real worktree. `--self-test fail` proves the
adapter detects a scratch checkout it cannot provision (a dependency shape
it doesn't recognize, for instance): non-zero exit, diagnostics on stderr.
Both modes route through the same provisioning logic normal mode uses —
never a parallel check that happens to agree with it on the fixture (§4's
general rule, the exact one finding #7 in `docs/tradeoffs.md` was written
after).

### 3.7 `migrate-rehearse` — eligible only on a migration-touching Class 2 task

`migrate-rehearse` sits in §3's "operates on nothing" row like `test` and
`mutate`, and — unlike `contract-check`/`ui-render`/`ui-conformance`/
`ticket-fetch`/`open-pr`/`worktree-prep` (§3.2–§3.6, §3.9) — it *is* invoked directly by
`floor`'s own dispatch loop, not by a skill's separate orchestration. But
it is not always-on the way `test` is: `floor` runs it only when **both**
hold —

- the class is 2 (`core/rules/migrations.md`: "`migrate-rehearse` is a
  Class 2 gate" — there is no Class-1 opt-in analogous to `mutate`'s
  `mutate_class1_optin`), and
- the task's own changed-file set (`compute_changed_files`, the same set
  every other diff-scoped capability reads, bookkeeping paths already
  excluded) contains a path under a `migrations/`- or `migration/`-named
  directory — the same `**/migrations/**` / `**/migration/**` glob
  `core/rules/migrations.md`'s own frontmatter uses.

A Class 2 task whose diff never touches a migration path never pays this
capability's cost, and never shows a gap for it — same as `ui-render`
against a diff that never touches a UI path (§3.3). A Class 2 task whose
diff *does* touch a migration path always accounts for `migrate-rehearse`
one way or another: `implemented` runs it as a normal pass/fail gate
(fail-fast, same as any other capability); anything else (`unavailable`,
`not-applicable`, or missing from `.spine/capabilities.json`) is recorded
as an explicit `degraded` result, the same disclosed-degradation shape
`clone-scan`/`callers`/`mutate` already use when unavailable (§1) — never
silently absent, and never a reason for `floor` itself to fail.

**Self-test** (§4): `--self-test pass` builds migrate → verify invariants
→ rollback → re-migrate against a throwaway seeded fixture inside its own
scratch space (never a real environment), all four steps succeeding;
`--self-test fail` does the same with a fixture where at least one step is
guaranteed to fail (e.g. a rollback that doesn't actually restore the
pre-migration shape), non-zero exit with full diagnostics.

**Restore-not-reset** (§2.2): `migrate-rehearse`'s own cleanup — on
success or on failure — must leave its scratch stack in whatever state it
found it in, not whatever state its own internals happen to produce at
exit. `--self-test pass` proves this too, not just the four migrate
steps: it runs the fixture twice — once starting from the scratch stack
already up (asserting it is *still* up afterward, i.e. never stopped) and
once starting from it down (asserting `migrate-rehearse` both started it
for its own run and stopped it again on the way out) — a single clean
`--self-test pass` covers both starting conditions.

### 3.8 `smoke-seed` / `smoke-run` / `smoke-golden` — a fixed sequence, each self-healing independently

These three run, always in this order, whenever Layer 5 smoke joins a
floor run (§7's `smoke_in_floor`, or unconditionally at Class 2): `floor`
seeds a smoke environment (`smoke-seed`), exercises it (`smoke-run`), then
checks its output against a golden result (`smoke-golden`) — fail-fast,
same as every other layer.

All three assume a running local stack, and per §2.2 none of them may
assume any other capability — inside this sequence or outside it — already
brought that stack up. Each is independently invocable (directly by a
human, by a skill re-checking one gate, by a resumed floor run), so each
must self-heal on its own: resolve whatever it needs from the stack (a
connection string, a service's own generated key, a running process) by
starting the stack first if it isn't already up — the same way `smoke-run`
already does — never fail with a "could not resolve X — is it running?"
diagnostic when starting the stack is within the adapter's own power. This
was the concrete gap a real `smoke-seed` shipped without: it resolved its
key via a status check with no restart fallback, so it deterministically
failed the first time anything earlier in the same `floor` run — including
`migrate-rehearse`'s own end-of-run cleanup — left the stack down,
independent of whether anything `smoke-seed` actually seeds was broken.

**Self-test**: alongside `--self-test pass`/`--self-test fail` (§4), each
of these three additionally supports a third scenario:

```
.spine/adapters/<name> --self-test cold
```

which builds the same passing fixture as `--self-test pass`, but starting
from its scratch stack torn down — proving the self-heal path above is
real, not just asserted. Same output discipline as `pass` (exit 0, one
line). Per §4's "exercise the real invocation path" rule, `--self-test
cold` must route through the same self-heal branch the adapter's normal
invocation uses, not a parallel check that starts the stack a different
way. `core/scripts/adapter-conformance` invokes `--self-test cold` for
these three capabilities specifically (§4) and fails conformance if it's
missing or non-zero — a `smoke-seed`/`smoke-run`/`smoke-golden` that
doesn't self-heal cannot stay `implemented`.

**The Class 2 hard gate and the M0 bootstrap waiver.** At Class 2, `floor`
fails if `smoke-run` is not `implemented` — smoke is a hard gate there, not
a soft skip. One narrow exception exists so milestone 0 can build smoke at
all: `floor` records `degraded:waived-bootstrap` (never a pass; smoke does
not run) instead of failing when **all** of these hold — the run has a
`--task` whose `work/<task-id>/milestone` is exactly `M0`; `capabilities.json`
has `smoke-run` `unavailable` with a reason containing "skeleton target";
and `work/M0/milestone.md`'s Capability targets row for `smoke-run` reads
`unavailable*`. Any other status shape (`missing`, `not-applicable`), any
other milestone, or a missing declaration keeps the hard gate. The waiver
lapses the moment `smoke-run` is `implemented`. See `docs/tradeoffs.md`,
"The M0 bootstrap waiver".

### 3.9 `ui-conformance` — declared tokens/components actually present, never visual similarity

Conditional existence, same shape as `ticket-fetch`/`open-pr`/
`worktree-prep`: only ever exists in a project with a UI handoff
bundle (§1). Sits in §3's "operates on nothing" row like `ui-render`: no
stdin, no positional argument, exit code alone governs pass/fail. The
adapter owns end-to-end enumeration and rendering, exactly the same
delegation §3.3 already uses for routes: for every `docs/ui/
screens/<id>.json` naming a `route`, render that route for real (an
adapter may share its dev-server bring-up with its own `ui-render`
adapter — core does not care how, only that each capability's own pass
criterion below is met) and check, deterministically:

- **Every entry in that screen's `components_used` list is actually
  present in the rendered output** — by whatever stable marker this
  project's component library exposes (a `data-*` attribute, a rendered
  class name, a component's own registered display name). Which marker to
  check is a stack-specific detail the adapter itself owns, same as
  `ui-render` already owns route-driving.
- **Key values from `docs/ui/tokens.json` this screen depends on
  resolve, in the real render, to what's declared** — not a stale or
  hand-typed substitute a diff introduced instead of reusing the token.

**Pass criterion**: every screen with a route in `docs/ui/screens/`
renders with every declared component present and every checked token
value matching. **This is a structural/identity check, not a
visual-similarity check** — it answers "did the build actually use what
the handoff declared," never "does it look right." A screenshot under
`docs/ui/screenshots/` (`core/templates/ui-handoff.md`) is
grounding material for the agent while it writes the code, and for a
human skimming the task's briefing — it is deliberately never the input
to *this* capability's pass/fail decision. Comparing a render against the
screenshots is a separate, reviewed (not scripted, not a floor gate)
step: `ui-capture` (§3.10) plus the `ui-fidelity` agent. Pixel- or perceptual-diffing a
real render against a golden screenshot is exactly the kind of gate that's
flaky across font rendering, anti-aliasing, and dynamic content without
dedicated image-diff infrastructure this core does not ship; a flaky
floor-adjacent gate erodes trust in every other gate next to it (the
"teeth, not more prompting" thesis README.md opens with), so this
capability stays deliberately narrower and fully deterministic instead of
reaching for that.

**Eligibility, not always-on**: same shape as `ui-render` (§3.3) — never
invoked by `floor`'s dispatch loop at all. `core/skills/verify/SKILL.md`'s
own orchestration runs `core/scripts/ui-touch` first (reusing the same
result §3.3's own check already produced this pass, never re-running it);
`ui-conformance` is only invoked when that reports a declared UI path
was touched, on a project where this capability is `implemented`, gated
by its own class opt-in (`ui_conformance_class1_optin`,
mirroring `ui_render_class1_optin`'s own default-`false` shape) — kept
separate from `ui_render_class1_optin` rather than reusing it, since a
team may want a real render checked (cheap, behavioral) without also
wanting design-token conformance enforced on every Class 1 change (a
stricter, more opinionated gate).

**Self-test**: `--self-test pass`/`--self-test fail` build their own
throwaway screen fixture (a tiny local page, plus a matching throwaway
`screens/<id>.json`/`tokens.json`/`components.md`) inside scratch space —
never the project's real bundle. `pass`: the fixture's render actually
contains its declared component and token. `fail`: the fixture is built
to be missing one on purpose, and the adapter's real check must catch it
— same "route both modes through the same check" rule §4 states generally.

### 3.10 `ui-capture` — real renders of every state, captured for review, never judged

Conditional existence, same shape as `ui-conformance` (§3.9): only exists in
a project whose `ui-render` is `implemented` *and* whose handoff bundle has
screenshots (`docs/ui/screenshots/`); otherwise `not-applicable` with reason
"no UI render capability or no reference screenshots in this project." Sits
in §3's "operates on nothing" row, plus one environment variable, following
the `SPINE_BASE_REF` / `SPINE_CONTRACT_NAME` precedent (§3.2), not a
positional argument:

| Variable | Carries |
|---|---|
| `SPINE_UI_CAPTURE_DIR` | Absolute path of the directory the adapter writes its captures into (`work/<task-id>/artifacts/ui-fidelity`, resolved by `/verify`). |

**What it does.** For every in-scope screen (the built-screens list below),
and for every state in that screen's `states[]` that has a
`screenshots[<state>]` entry, render the route for real, drive it into that
state, and write under `$SPINE_UI_CAPTURE_DIR/<screen-id>/`:

- `<state>.render.png` — screenshot comparable to the reference. For a
  full-screen reference, a page screenshot at the reference's pixel
  dimensions. For a **crop** reference (a popover, drawer or modal image much
  smaller than the screen), a screenshot of that overlay's own element, so the
  pair is like-for-like; coverage records `reference_kind: "full" | "crop"`
  and both images' dimensions. If the overlay element can't be resolved, the
  state is `driver_failed`, never a mismatched full-page render;
- `<state>.reference.png` — a copy of `screenshots[<state>]`;
- `<state>.facts.json` — per component element the project's marker
  convention exposes (the same marker `ui-conformance` checks): name,
  variant if exposed, bounding rect, computed fill/border/text color/font,
  visible text; plus the viewport, and every declared `components_used[]`
  entry (with props) that had no rendered node (`declared_missing`);
- `spec.json` — the screen's `docs/ui/screens/<id>.json`, verbatim;
- `coverage.json` — every state in `states[]` with a status:
  `captured`, `no_screenshot` (declared, no image), `no_driver` (image, but
  no way to reach the state), or `driver_failed` (a step could not find its
  target — the message is recorded).

**Authentication and state driving.** Both are stack-specific details the
adapter owns, like route-driving in §3.3. A screen that needs a session is
captured in a signed-in browser context; the adapter reuses whatever
signed-in context the project's own `ui-render`/`ui-conformance` establish
rather than building a second one, and must never rely on a test-only auth
bypass. Reaching a non-default state is done one of two ways, recorded per
state in `coverage.json` as `driver: "real" | "gallery" | "steps"` (`real` = the signed-in route itself): (a) a project
**state gallery** — a development-only route rendering a screen's view for a
named state from typed test data (e.g. `/__ui/<screen>?state=<state>`),
preferred where it exists because it needs no interaction scripting; or (b) a
per-screen `docs/ui/states/<id>.json` mapping each state name to ordered
steps (`click`/`fill`/`press`, addressed by role and name or visible text,
plus an optional `wait_for` text) run from a fresh load of the route.
`default` should be captured from the real, signed-in route, not the
gallery, so at least one state per screen proves the shipped screen renders.
A gallery state renders test data, not the live endpoint — it shows the view
can look right in that state, not that the app reaches it.

**Pass criterion.** Exit 0 means every in-scope screen's route rendered and
`coverage.json` was written for it. Non-zero means a capture itself failed
(route did not render, browser error, no capture dir). `no_screenshot`,
`no_driver` and `driver_failed` are **recorded, not exit-code failures** —
the reviewer reports them. **This capability never judges fidelity.** The
judgment is `ui-fidelity` (`core/agents/ui-fidelity.md`), dispatched by
`/verify` step 1e, whose findings must cite a captured fact as `render`
evidence (§5) — a judgment call cannot honestly be a self-testable boolean,
and this is why the deterministic capture and the model's review are split.

**Scope: the built-screens list.** `.spine/ui-built-screens.txt`, when
present and non-empty, names one `screen_id` per line
(`docs/ui/screens/<id>.json`) and limits `ui-render`, `ui-conformance` and
`ui-capture` to those screens; absent or empty means every screen with a
spec. This is a contract convention shared by all three, not a per-adapter
one — a screen is added to the file when a task builds it.

**Eligibility, not always-on.** Never invoked by `floor`. `/verify` step 1e
runs it only when `ui-touch` reports a declared UI or content path was
touched, gated by its own class opt-in `ui_fidelity_class1_optin` (default
`false`; Class 2 always eligible), separate from the other two UI opt-ins
because it is the costliest and most opinionated of the three (it runs a
multimodal review per screen).

**Self-test**: `--self-test pass`/`--self-test fail` build a throwaway page,
throwaway screen spec, 1x1 reference PNG per state and a state-step file
with one click-reached state inside scratch space, and run the same capture
path normal mode uses. `pass`: PNG magic bytes present, `facts.json` per
state lists the marked components with non-zero rects, and the clicked
state's facts differ from `default`'s. `fail`: a state step targets a
control that does not exist (must exit non-zero or record `driver_failed`
distinctly), and separately an empty-body route must exit non-zero. It
proves the capture works, not that any UI is right.

## 4. The self-test convention (what makes conformance possible)

Every adapter additionally supports exactly one more invocation shape,
mutually exclusive with normal operation:

```
.spine/adapters/<name> --self-test pass
.spine/adapters/<name> --self-test fail
```

(`smoke-seed`, `smoke-run`, and `smoke-golden` additionally support a
third scenario, `--self-test cold` — §3.8; §2.2's shared-environment-state
rule they exist to prove is the only case today that needs a third mode.)

In `--self-test pass`, the adapter builds or selects a fixture *inside its
own temp scratch space* (never the project's real files) that is guaranteed
to satisfy the capability, runs itself against it, reports exit 0 with the
one-line-success discipline, and cleans up before returning.

In `--self-test fail`, the adapter does the same with a fixture guaranteed to
violate the capability, and reports non-zero with full diagnostics.

**For a changed-file-set capability** (§3's second row — `test-changed`,
`clone-scan`, and, since Phase E, `lint`/`typecheck`), `--self-test pass`
must additionally prove the scoping itself, not just the underlying check:
the fixture must include a second, violating artifact that is *not* part
of the stdin set fed to the self-test run, and the adapter must still
report a clean pass. A self-test that only ever builds one clean file
proves the tool wrapper works; it proves nothing about whether the
adapter actually honors the changed-file scope versus silently checking
the whole scratch directory regardless of stdin. This is the fixture that
would have caught a rescoped-in-name-only adapter — see
`docs/tradeoffs.md`'s Phase E section for the concrete horizon/synthetic
runs that exercised it.

This is a builder decision — it's the mechanism that makes `adapter-conformance` possible
without inventing a second, stack-specific fixture-delivery channel. An
adapter author validating a hand-written adapter gets the same self-test
modes `adapter-conformance` uses; there's only one path to "conformant."

**The self-test must exercise the adapter's real invocation path, not a
parallel check that happens to agree with it on the fixture.** A self-test
that re-implements its own ad-hoc pass/fail logic (e.g. a second, simpler
grep or comparison inlined in the `--self-test` branch) instead of calling
through the same function/command the normal-mode branch calls proves
nothing about the adapter's actual behavior — it can pass forever while
the real path is broken on inputs the toy check never represents (a real
testbed finding: a `callers` adapter's self-test grepped a bare
symbol name against a fixture, while the real adapter greps a full
repo-relative file path against real source — a shape neither the pass nor
the fail fixture ever exercised, so a heuristic that structurally could
not match any real relative or aliased import stayed "conformant"
indefinitely). Build the fixture, then invoke the same code path normal
mode uses against it — never a second implementation of the check.

**Both of the above are authoring requirements, not mechanically-checked
guarantees.** `adapter-conformance` validates `--self-test pass`/`fail`'s
exit code and one-line-output shape; it does not inspect whether a
scoping fixture genuinely includes an out-of-scope violator, or whether a
self-test branch calls through the adapter's real invocation path rather
than a parallel check — the `callers` incident above is exactly a case
that satisfied every check `adapter-conformance` runs while violating this
one. A human reviewing a hand-written adapter is the real backstop for
these two rules (see `docs/tradeoffs.md`'s Known limits).

**A `--self-test fail` branch with two divergent internal paths needs a
way to say which one fired.** Some fail fixtures don't just build a
single guaranteed-violating case — they first drive the adapter's real
check against a fixture *expected* to fail it, and only reach the generic
fail line if that attempt itself behaves as expected (e.g. a fixture that
drives a flow which should time out because nothing was ever seeded for
it; if the flow unexpectedly succeeds, that's not "the fixture correctly
failed," it's a hole in the check being self-tested). Left alone, both
paths — "correctly demonstrated the failure" and "the fixture's own
unexpected-success branch fired" — converge on the same generic
`exit 1`, and `adapter-conformance` only ever checks `fail_code -ne 0`
plus non-empty diagnostics, so a regression that broke the underlying
check into always-succeeding would still read as a clean, conformant
`--self-test fail`. Only a human reading stderr text would ever notice.

The convention: a `--self-test fail` branch whose own unexpected-success
path fires must print a line starting with `SELF-TEST-FAIL-FIXTURE-BROKEN:
<reason>` to stderr before its `exit 1` (in addition to whatever ordinary
diagnostics that path already prints). `adapter-conformance` greps
`--self-test fail`'s combined output for that literal prefix; finding it
is reported as its own distinct `FAIL` — the underlying check has a real
hole — never folded into the generic `ok` a correctly-demonstrated
failure gets. This is additive, not a new requirement on every fail
branch: a fail fixture that only ever builds one guaranteed-violating
case, with no "did the check even catch it" sub-probe, has nothing to
distinguish and never needs the marker — omitting it degrades to
today's behavior (an ordinary, undifferentiated fail), never a false
failure. Adopt it only where a fail branch's own internal probe
succeeding or failing is itself part of what's being asserted.

Design note this implies: an adapter must be able to construct at least one
concrete pass case and one concrete fail case for its own capability, fully
self-contained. Where a capability's tool has nothing meaningful to
self-test (e.g. it always trivially passes/fails independent of input), that
is itself a sign the capability should be `unavailable` or `not-applicable`
rather than `implemented` — conformance existing to prove the teeth are real
(`core/scripts/adapter-conformance` on an always-passing adapter must fail
its own suite by construction).

## 5. Verdict schema (adversary output, validated by `verdict-filter`)

Falsifier and security adversary subagents (Phase C), and the `ui-fidelity`
reviewer (§3.10), write one JSON object per run:

```json
{
  "agent": "falsifier",
  "task_id": "20260101-example",
  "attacked": [
    "acceptance check 3 against a concurrent-write input",
    "invariant: org-scoped reads, per caller map entry X"
  ],
  "verdicts": [
    {
      "claim": "string, required, non-empty",
      "severity": "high | medium | low",
      "evidence": {
        "kind": "file_line",
        "file": "src/foo",
        "line": 42
      }
    },
    {
      "claim": "another finding",
      "severity": "medium",
      "evidence": {
        "kind": "command",
        "command": ".spine/adapters/test --stub-feature",
        "output": "captured output proving the claim"
      }
    }
  ]
}
```

`attacked` is mandatory and non-empty even when `verdicts` is empty — it's
the clean-bill enumeration ("what was attacked") this schema requires.
`evidence.kind` is `file_line` (requires non-empty `file` and integer
`line`), `command` (requires non-empty `command` and `output`),
`render` (`ui-fidelity` only — requires non-empty `screen_id`, `state`,
`artifact` and `quote`: `{"kind":"render","screen_id":"<id>","state":
"<state>","artifact":"<path relative to the capture directory>","quote":
"<verbatim span>"}`; `verdict-filter`'s pass 2 requires `artifact` to
resolve to a real file inside `work/<task-id>/artifacts/ui-fidelity/` (no
absolute path, no `..`) and `quote` to appear verbatim in it — a model may
misread a captured fact, but cannot cite one the capture never recorded; a
visual difference no captured fact expresses has no valid evidence shape and
is dropped, deliberately trading recall for trust), or
`decision` (design-stage extension, Extension A — requires non-empty
`decision_id` and `quote`: `{"kind":"decision","decision_id":"D-<n>",
"quote":"<verbatim span>"}`). Any verdict missing a required field, with an
empty claim, an unrecognized severity, or malformed evidence is **dropped**
by `verdict-filter` before `/verify` ever reads the file — a script decides
what counts as evidence, never a model. `decision` evidence gets a second,
stronger check beyond shape: `decision_id` must resolve to exactly one real
`docs/decisions/<id>-*.md` record and `quote` must appear verbatim in it
(`core/scripts/verdict-filter`'s own pass 2) — a decision id is trivially
cheap to fabricate compared to a real `file:line` or a real command's
captured output, so it earns the extra check.

**`disposition` (written by `/verify`, never by the adversary itself)**: once
a kept verdict has been acted on — fixed, or deliberately left for later —
`/verify` §5 adds `"disposition": "fixed" | "not_fixed"` to that verdict
object in `work/<task-id>/artifacts/<agent>-verdict.json`, the same moment
it decides how to word the matching bullet in `verify.md`'s prose (no new
judgment, one more field recording a decision already made). Absent means
`not_fixed` — a verdict `/verify` never got around to marking must never
silently read as resolved. The adversary's own raw output never sets this
field; `verdict-filter` passes it through unmodified either way, since it
validates verdict shape at dispatch time, before any fix decision exists.
This is what lets `/ship` (`core/skills/ship/SKILL.md` §3a — flagged-
finding triage) identify kept-but-unresolved findings mechanically instead
of re-parsing
`verify.md`'s free prose, which uses different wording for the same
outcome from one task to the next.

**Cross-repo verdicts (Extension B)**: a falsifier run against a multi-repo
task's diff (`core/agents/falsifier.md`'s "Cross-repo mandate") cites
undeclared coupling using the existing `file_line` kind, no new evidence
kind needed — only `evidence.file` is repo-qualified (`"<repo-name>:
<path>"`, the same convention `## Predicted touch` already uses for
multi-repo plans) so the finding is unambiguous across repos.
`verdict-filter` does not parse or validate that qualification; it only
checks non-emptiness, exactly as it already does for a single-repo
`file_line`.

### 5.1 Shared adversary discipline (falsifier, security)

The behavioral discipline below governs both adversary agents identically
— `core/agents/falsifier.md` and `core/agents/security.md` each carry only
a short pointer to this subsection (themed to "scenario"/"attack" as their
own vocabulary needs) rather than their own copy of this prose. This is
the canonical text; if the two agent files ever again show near-identical
paragraphs instead of a pointer here, that's the exact duplication-drift
this subsection exists to prevent — collapse it back to one copy, here.

**Reporting discipline.** Every verdict needs evidence §5 above will
accept: a `file:line` pair, or a command plus its actual captured output —
not a description of what a command would probably show. A claim without
one of those two evidence shapes gets dropped by `verdict-filter` before
anyone reads it, so don't bother filing it; strengthen it or drop it
yourself.

**Verify each piece of evidence once.** Read the source, note the exact
line/quote, and move on — do not re-run overlapping greps/seds against a
span you've already confirmed matches, and never re-check the same quote
twice looking for more confidence. If a quote won't match cleanly on the
first check, shorten it to a shorter unambiguous span rather than
iterating on the same one. The JSON reply is the deliverable;
re-verification that can't change your answer only delays it.

**Be efficient.** Reach a conclusion and act on it rather than extensively
deliberating before each step — construct the case, check it, write the
verdict, move to the next one. Prolonged internal reasoning before acting
is not a substitute for more ground covered; when in doubt, spend the time
on one more attack rather than re-weighing one you've already decided.

**Reply shape**: your entire reply must be exactly one JSON object,
nothing before or after it — the caller writes your reply verbatim to a
file and runs it through `verdict-filter`. The example JSON block earlier
in §5 is the exact shape both agents match; the only fields either agent
file needs to state itself are `"agent"` (the literal string `"falsifier"`
or `"security"`) and what belongs in `attacked` (each agent's own mandate
determines that list's real content — see each file's own text). `verdicts`
may be empty; `attacked` may never be.

## 6. Task-ID and commit trailer convention

Task IDs are `<YYYYMMDD>-<kebab-slug>`, e.g. `20260807-shared-unit-types`.
The work folder is `work/<task-id>/`. Every commit produced by `/ship` for
that task carries the trailer:

```
Spine-Task: <task-id>
```

on its own line in the commit message footer, composing cleanly alongside
any other trailer (e.g. `Co-Authored-By:`) and any Conventional-Commits-style
subject prefix a project's own CI enforces — the trailer says nothing about
commit-message grammar and never fights it.

This makes the untracked-commit ratio mechanical to audit by hand: a
`git log` trailer grep over a window can report commits carrying none of
`Spine-Task:`, `Spine-Ticket:`, or `Spine-Bypass:` as off-spine work, by
definition — no model judgment involved. `/ship --bypass <reason>` writes
`Spine-Bypass: <reason>` instead of (or alongside) the task trailer, so
bypassed work is still mechanically visible and distinguishable from
silent drift.

**Class 0 (traced-trivial) and the `Spine-Ticket:` trailer.** A Class 0 change
has no task folder and no `Spine-Task:` id, but it is not off-spine: the
traced-trivial path (`core/skills/task/SKILL.md` §1) commits it carrying

```
Spine-Ticket: <ticket-key>
```

— the key parsed directly from the current branch/commit convention, never
invented. Spine does **not** enforce this trailer with a hook: the
surrounding org already requires a ticket on every commit, so a second gate
would be redundant — a spine value `/ratchet` and the stack-independence
rule both reject. The trailer is spine's own convention so a trailer grep
can distinguish a traced Class 0 commit from genuinely off-spine work.
**Every `/ship` commit (Class 1/2) also carries `Spine-Ticket:
<key>` alongside its `Spine-Task:` trailer when a ticket is available** — same
derivation, same composing rule; the spine task id and the ticket key travel
together. When no ticket is derivable (genuinely off-ticket), the trailer is
omitted and the commit counts as off-spine as before.

**Multi-repo tasks (Extension B)** use the *same* task ID — generated once,
at the workspace root, per `core/skills/task/SKILL.md` — as the
`Spine-Task:` trailer on every repo's own commit for that task. A task
touching two repos produces two commits (one per repo, in the declared
ship order, `core/skills/ship/SKILL.md`'s staged-ship §), both carrying an
identical `Spine-Task: <task-id>` trailer. This is spine's existing
linkage primitive doing the cross-repo join — a trailer grep
run against any one member repo's own `git log`
still works unmodified, since it only ever inspects that repo's own
commits for the trailer's presence; it does not need to know a commit's
trailer also appears in a sibling repo.

## 7. Team strictness profiles (`.spine/profile.json`, Phase 5)

A profile is the one place a *team* sets its default strictness, so several
teams can adopt one spine at different defaults without re-interviewing each
engineer. It is committed, repo-level project state — the same shape as
`.spine/capabilities.json` and `.spine/protected-paths.conf`. Absent = the
built-in `standard` defaults apply. Fields, all optional (an absent field
takes its `standard` value):

| Field | Values | Governs |
|---|---|---|
| `profile` | `prototype` \| `standard` \| `regulated` \| `custom` | Label only (which preset this started from). |
| `class1_adversaries` | `1` \| `2` | How many adversaries `/verify` runs on a Class 1 task (`1` = falsifier only; `2` = falsifier + security). Never `0` — the falsifier always runs. |
| `smoke_in_floor` | `true` \| `false` | Whether smoke joins the floor when its runtime fits the budget. |
| `class0_max_files` | `0`..`10` | The Class 0 (trivial) file-count threshold `/intake` and `/task` classify against. |
| `class0_max_lines` | `0`..`100` | The Class 0 line-count threshold. |
| `autonomy_ceiling` | `guided` \| `checkpointed` \| `auto` | The highest autonomy a Class 1 task may run at (`core/skills/task/SKILL.md`). Class 2 is always `guided` regardless. |
| `pr_open` | `never` \| `auto-only` \| `auto-checkpointed` \| `all` | When `/ship` opens a draft PR (`§3.5`, `core/skills/ship/SKILL.md` §5a). `auto-checkpointed` (default) opens for `auto`/`checkpointed`; `all` also opens for `guided`; `auto-only` only for `auto`; `never` leaves every PR to the human. |

**Presets** (a starting point `/bootstrap`/`/adopt` write, then the team edits):

| | prototype | standard | regulated |
|---|---|---|---|
| `class1_adversaries` | 1 | 2 | 2 |
| `smoke_in_floor` | false | true | true |
| `class0_max_files` / `_lines` | 3 / 30 | 2 / 15 | 1 / 10 |
| `autonomy_ceiling` | auto | auto | checkpointed |
| `pr_open` | all | auto-checkpointed | auto-checkpointed |

**The hard invariant, mechanically enforced by `core/scripts/profile-check`**
(run at every `setup --check`, i.e. every `/task` step 0): a profile tunes
*ceremony* and *stops*; it can **never** disable the mechanical floor, the
protected-path hook, or the autonomy-caps-class rule. Those are not fields in
the schema — they are *unrepresentable* — and `profile-check` rejects any
unknown key (so a hand-edited `"floor": false` is caught) and any out-of-range
value (adversaries `< 1`, an oversized `class0_max_*` that would make
substantial changes "trivial"), the same fail-closed way `adapter-conformance`
rejects a miscalibrated adapter. An invalid profile fails `setup --check` loudly
rather than silently enforcing something no one chose.

**Precedence**: for the two fields that overlap Layer 1 user-config
(`class1_adversaries` ↔ `ceremony.class1_adversary_count`, `smoke_in_floor` ↔
the smoke budget), the **team profile wins** when present — the point of a
profile is one consistent standard per team regardless of who is working.
Layer 1 remains for genuinely personal preferences. A field absent from the
profile falls back to Layer 1, then to the built-in default.
