# Adapter contract

Authoritative for every file in `core/scripts/`, every adapter in a project's
`.spine/adapters/`, and `core/scripts/adapter-conformance`. Nothing here names
a language, framework, or tool — this document, like the core, is stack-blind.

## 1. What a capability is

A capability is an executable at `.spine/adapters/<name>` in the installed
project. The core never calls a tool directly; it calls a capability name.
Eighteen capabilities exist:

`typecheck`, `lint`, `test`, `test-changed`, `secret-scan`, `dep-diff`,
`clone-scan`, `callers`, `mutate`, `smoke-seed`, `smoke-run`, `smoke-golden`
(§3.8), `migrate-rehearse` (§3.7), `contract-check` (Extension B — §3.2),
`ui-render` (§3.3), `ticket-fetch` (intake — §3.4), `open-pr` (ship —
§3.5), `worktree-prep` (falsifier — §3.6).

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

`ui-render` only ever exists in a project whose runtime shape serves a
browser UI a person looks at (`core/skills/bootstrap/SKILL.md` /
`core/skills/adopt/SKILL.md`'s Layer 3 calibration question) — a project
with no browser UI (a CLI, a library, a pure API with no rendered surface)
marks it `not-applicable` with reason "no browser UI in this project's
runtime shape." Like `contract-check`, `ui-render` is never invoked by
`floor`'s dispatch loop — `core/skills/verify/SKILL.md`'s own orchestration
invokes it directly, and only when `core/scripts/ui-touch` reports the
task's diff actually touched a UI-file-shape path (`.spine/ui-paths.conf`).
See §3.3.

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
`verify.md`, the ledger, and the delta briefing.

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

### 3.3 `ui-render` — a real browser, not a mocked render

Every other capability that touches UI code (`test`, `test-changed`) runs
against a mocked DOM or mocked network layer — real, valuable coverage,
but structurally blind to a class of bug that only shows up when the
actual view is served by the actual dev server and rendered in an actual
browser: a network call that returns something other than what the test
mocked (a dev-server SPA fallback returning 200+HTML for an unmatched API
route, for instance), a CSS/layout failure invisible to jsdom/happy-dom, a
route that 404s for real. `ui-render` exists to catch exactly this gap, not to replace or
duplicate `test`/`test-changed`.

`ui-render` sits in §3's "operates on nothing" row: no stdin, no
positional argument, exit code alone governs pass/fail (§2). The adapter
is responsible end-to-end for standing up whatever it needs to render a
real page — starting the project's own dev server (or reusing one already
running), driving a real or headless browser to the project's real
routes, and tearing down anything it started before returning, pass or
fail. Which routes to drive and how to reach a rendered page is a
stack-specific detail the adapter itself owns (e.g. reading a small
project-local routes list it maintains, or the same entry points
`docs/map.md`'s survey already names) — core never enumerates routes on
the adapter's behalf.

**Pass criterion, at minimum**: every route the adapter drives renders a
non-empty, non-whitespace body, and (where the adapter can determine it)
the rendered content is one of that route's own known states — not a
generic framework shell, not an empty successful-request placeholder. An
adapter that only checks "the page returned a 200" has not implemented
this capability correctly; the entire reason it exists is to catch a
blank/wrong render that a 200 status code would not.

**Eligibility, not always-on**: unlike `test`, `ui-render` is not run
unconditionally by `floor` — it is never invoked by `floor`'s dispatch
loop at all. `core/skills/verify/SKILL.md`'s own orchestration runs
`core/scripts/ui-touch` against the task's real diff first; `ui-render`
is only invoked when that reports the diff touched a UI-file-shape path
(`.spine/ui-paths.conf`, project-declared, written at `/bootstrap`/`/adopt`
time). A task whose diff never touches a view/component file never pays
this capability's cost.

**Self-test**: `--self-test pass`/`--self-test fail` build their own
throwaway page/fixture inside scratch space and a throwaway static server
to serve it — never the project's real dev server or real routes, same
isolation every other capability's self-test already requires (§4). Both
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
`mutate`, and — unlike `contract-check`/`ui-render`/`ticket-fetch`/
`open-pr`/`worktree-prep` (§3.2–§3.6) — it *is* invoked directly by
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

Design note this implies: an adapter must be able to construct at least one
concrete pass case and one concrete fail case for its own capability, fully
self-contained. Where a capability's tool has nothing meaningful to
self-test (e.g. it always trivially passes/fails independent of input), that
is itself a sign the capability should be `unavailable` or `not-applicable`
rather than `implemented` — conformance existing to prove the teeth are real
(`core/scripts/adapter-conformance` on an always-passing adapter must fail
its own suite by construction).

## 5. Verdict schema (adversary output, validated by `verdict-filter`)

Falsifier and security adversary subagents (Phase C) write one JSON object
per run:

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
`line`), `command` (requires non-empty `command` and `output`), or
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

This makes the untracked-commit ratio mechanical:
`core/scripts/ledger scan-untracked-ratio` greps `git log` trailers over a
window and reports commits carrying none of `Spine-Task:`,
`Spine-Ticket:`, or `Spine-Bypass:` as off-spine work, by definition — no
model judgment involved. `/ship --bypass <reason>` writes `Spine-Bypass:
<reason>` instead of (or alongside) the task trailer, so bypassed work is
still mechanically visible and distinguishable from silent drift.

**Class 0 (traced-trivial) and the `Spine-Ticket:` trailer.** A Class 0 change
has no task folder and no `Spine-Task:` id, but it is not off-spine: the
traced-trivial path (`core/skills/task/SKILL.md` §1) commits it carrying

```
Spine-Ticket: <ticket-key>
```

— the key derived from the branch/commit convention (`ledger
ticket-from-branch`), never invented — and appends a one-line in-project record
via `ledger trace` (to `.spine/trace.jsonl`). Spine does **not** enforce this
trailer with a hook: the surrounding org already requires a ticket on every
commit, so a second gate would be redundant — a spine value `/ratchet` and the
stack-independence rule both reject. The trailer is spine's own convention so
`scan-untracked-ratio` can distinguish a traced Class 0 commit from genuinely
off-spine work, and so the dashboard can surface trivial work that tracker linkage
alone never would. **Every `/ship` commit (Class 1/2) also carries `Spine-Ticket:
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
linkage primitive doing the cross-repo join — `ledger
scan-untracked-ratio` run against any one member repo's own `git log`
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
