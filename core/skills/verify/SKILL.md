---
name: verify
description: Run the deterministic floor and the adversary agents for a task, and assemble verify.md. Invoked by /task at the verify phase — not a general code-review command.
disable-model-invocation: true
argument-hint: <task-id>
---

You are running `/verify` for task `$ARGUMENTS` (the task ID). Scripts live
at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`, the template at
`${CLAUDE_SKILL_DIR}/../../templates/verify.md`. Hand
`${CLAUDE_SKILL_DIR}/../../...` to the shell verbatim, `../../` included —
do **not** lexically collapse it to `.claude/`; `.claude/skills/verify` is
a symlink into the spine core checkout, and collapsing the text yields a
nonexistent `.claude/scripts/...` path. Read `work/<task-id>/class`
first — everything below branches on it.

**Multi-repo detection (Extension B)**: if `workspace.json` exists at the
project root, this is a workspace-root session and `work/<task-id>/plan.md`'s
`## Predicted touch` entries are repo-qualified (`<repo-name>:<path>`) —
every step below that mentions "per repo" applies; **a project with no
workspace.json runs every step below exactly as it always has, once,
against itself** — this is the zero-behavioral-change guarantee at the
skill level, not just the hook level. Derive the **edited repos** set from
`## Predicted touch`'s repo-name prefixes (the repos this task actually
changed) before step 1 — this is the set that gets a full floor; every
other repo a touched contract puts in blast radius gets `contract-check`
only, never a floor (the edited-vs-affected rule — a consumer's own
pre-existing, unrelated failures must never block a
producer task forever).

**Tooling-gap discipline applies to every script below** — see
`core/skills/task/SKILL.md`'s own header section for the three-outcome
rule (ran-passed / ran-failed / could-not-run) and how to record it
(notes.md). Several capabilities here need a specific default when they
could not run at all, because "silently treat as pass" is exactly the
failure mode this fix exists to close:

- **`floor` could not run at all** (distinct from `floor` running and
  reporting `FAIL`): this is not a `FAIL` you can fix by touching the
  diff, and it is not a `PASS` — nothing was checked. Record the gap,
  still assemble `verify.md` (§5) with `Floor: COULD NOT RUN` in place of
  the results table, and report `FAIL` to the caller with reason "floor
  could not run — see verify.md Tooling gaps." Never let an unrunnable
  floor read as a passing one.
- **`verdict-filter` could not run** for an adversary's raw verdict: fall
  back to the manual check it would have done — read the raw JSON by hand
  against `ADAPTER-CONTRACT.md §5`'s schema (non-empty `attacked`, every
  verdict carrying a valid `file_line` or `command` evidence shape) before
  using any of it in `verify.md`, and say explicitly in that section that
  filtering was manual, not mechanical, for this run.
- **`conformance` could not run**: record the gap with exactly this
  consequence — "conformance score unavailable — plan predictiveness
  unmeasured for this task."
- **`ui-touch` could not run at all**: do not treat this as "no UI
  touched" (that would silently skip step 1c's gate on a task that
  genuinely touched UI code) — record the gap with the consequence "UI
  render check eligibility unmeasured for this task; step 1c skipped
  without proof this diff didn't need it" and skip step 1c, loudly, rather
  than silently defaulting to not-touched.


**Voice.** Every message this skill leaves for the human follows
`core/templates/human-touchpoint.md`. Before sending one, run its "Before you
send" list: gloss or drop internal names, IDs and commit hashes, keep one
decision per message, and put surprises first.

## 1. Floor

Single-repo:

```
${CLAUDE_SKILL_DIR}/../../scripts/floor <class> --task <task-id> \
  --out work/<task-id>/artifacts/floor-result.json
```

**Multi-repo**: run this once per repo in the *edited* set, each with
`--project <repo-abs-path>` (from `workspace.json`) so it runs against that
repo's own `.spine/adapters/` and its own git history — `floor` itself is
unmodified (the edited-vs-affected rule lives in `/verify`'s orchestration,
not in `floor`'s own dispatch). `--task <task-id>` makes
`floor` write `callers.md`/`dep-diff.md` to *that repo's own*
`work/<task-id>/artifacts/` (verified for real: `floor` resolves this path
from its own `--project`, never from `--out` — no repo-name suffix needed
or produced, since `~/repoA/work/<task-id>/artifacts/callers.md` and
`~/repoB/work/<task-id>/artifacts/callers.md` are already distinct paths
by construction). Only `--out` itself (the floor-result JSON, whatever
path you give it) needs an explicit repo-distinguishing name if you choose
to collect them all in one place — `work/<task-id>/artifacts/
floor-result-<repo-name>.json` at the **workspace root** is the
recommended convention, so the aggregation in §5 has one place to read
every repo's pass/fail summary from, while the fuller `callers.md`/
`dep-diff.md` content the adversaries read stays wherever `floor` actually
put it: each repo's own `work/<task-id>/artifacts/`.

This fail-fasts on the first failing capability (per repo, in multi-repo
mode — one repo's floor failing doesn't stop another repo's floor from
still running and being recorded) and, for `--task`, writes
`callers`/`dep-diff` artifacts under `work/<task-id>/artifacts/` — the
adversaries read those, not the raw diff themselves. If any repo's floor
fails, still assemble `verify.md` (§5) so the failure and everything
reached before it is on record, then report FAIL to the caller. Don't run
the adversaries against a diff a deterministic floor already rejected —
that's wasted adversary budget on something that's going to change anyway.

## 1b. Contract conformance (multi-repo only, skip entirely otherwise)

```
${CLAUDE_SKILL_DIR}/../../scripts/contract-touch --project <workspace-root> \
  --out work/<task-id>/artifacts/contract-touch.json
```

For each `touched_contracts[]` entry:

- **Breaking-change gate.** If `spec_change == "breaking"`, read
  `plan.md`'s `## Contract change` line. Anything other than `expand` or
  `contract` there: this task fails verify outright — "breaking contract
  change without an expand/contract milestone step declared, see
  core/rules/contracts.md" — regardless of what the plan's prose claims
  elsewhere. This is a deterministic check; it reads the real diff's
  classification, never the plan's self-report alone.
- **Registry staleness.** If `registry_stale` is true, surface it in
  `verify.md` verbatim (§5) — this is a warning about `contract-touch`'s
  own blast-radius detection possibly being wrong, not a pass/fail signal
  by itself, but it must never be silently dropped (core/rules/
  contracts.md).
- **Gate the producer and every consumer, always** — `contract-check`
  eligibility is not the same thing edited-vs-affected governs; it governs
  *floor* eligibility only (step 1). A repo party to a touched contract
  gets `contract-check` whether or not this task edited it: an edited
  repo gets its `contract-check` result *in addition to* its full floor
  (step 1) — this is what makes "contract-check gating the consumer" true
  even in the additive worked example, where the consumer is directly
  edited in the same task — while an affected-only repo (not in step 1's
  edited set) gets `contract-check` **and nothing else**, never a floor.
  For the producer plus every name in `consumers`: check
  `.spine/capabilities.json` in that repo for `contract-check`'s status.
  If `implemented`, run it directly — **not through `floor`, even for an
  edited repo** —
  `SPINE_CONTRACT_NAME=<name> SPINE_CONTRACT_SPEC_PATH=<workspace-root>/<spec_path>
  .spine/adapters/contract-check` (CWD at that repo's root, per
  `core/ADAPTER-CONTRACT.md §3.2`). Record pass/fail exactly like a floor
  capability, in its own "Contract conformance" section (§5) — never
  folded into that repo's own floor table, even when both ran for the same
  repo. If not `implemented`, record it degraded with its recorded
  reason — same "never silently skip a gate" discipline as every other
  capability. A consumer gated this way never also runs its own floor for
  this task — that's the entire point of the edited-vs-affected rule.

## 1c. UI render check (only when the diff touches a declared UI path)

```
${CLAUDE_SKILL_DIR}/../../scripts/ui-touch --project <project root> \
  --out work/<task-id>/artifacts/ui-touch.json
```

Single-repo by default; multi-repo runs this once per repo in the
*edited* set (same set step 1 already derived), `--project <repo-abs-path>`,
each writing its own `work/<task-id>/artifacts/ui-touch-<repo>.json`.

If `ui_touched` is `false` for every repo checked: skip the rest of this
step entirely — no `ui-render` invocation, no section in `verify.md` (same
omit-whole-section rule step 1b's "Contract conformance" already uses when
nothing was touched — this is not a degraded or skipped gate, it's a gate
that correctly never applied).

If `ui_touched` is `true` for any repo: check whether this class is
eligible to run `ui-render` at all, same gating shape `core/scripts/floor`
already uses for `mutate`:

```
jq -r '.ui_render_class1_optin // false' ~/.spine/user-config.json
```

Class 2: always eligible. Class 1: eligible only if the above reads
`true` (default `false` — same opt-in-only default `mutate_class1_optin`
uses). Not eligible: record this plainly in `verify.md`'s own "UI render"
section as `SKIPPED (Class 1, ui_render_class1_optin not set)` — this is
neither a pass nor a capability gap, it is a deliberate calibration choice,
and must read as one, not as an unexplained absence.

Eligible: check `.spine/capabilities.json` for `ui-render`'s status in
that repo. If `implemented`, run it directly — **not through `floor`** —
`.spine/adapters/ui-render` (CWD at that repo's root, per
`core/ADAPTER-CONTRACT.md §3.3`). Record pass/fail in its own "UI render"
section (§5) — never folded into the floor table, same discipline as
"Contract conformance." If not `implemented` (`unavailable`/
`not-applicable`), record it degraded with its recorded reason — same
"never silently skip a gate" discipline as every other capability.

## 1d. UI conformance check (only when the diff touches a declared UI path)

Reuses the exact `ui-touch` result step 1c already produced this pass —
never a second invocation. If `ui_touched` was `false` for every repo
checked: skip this step entirely, same omit-whole-section rule step 1c
itself uses.

If `ui_touched` was `true` for any repo: check class eligibility, same
opt-in shape step 1c uses, its own separate key:

```
jq -r '.ui_conformance_class1_optin // false' ~/.spine/user-config.json
```

Class 2: always eligible. Class 1: eligible only if the above reads
`true` (default `false`). Not eligible: record `SKIPPED (Class 1,
ui_conformance_class1_optin not set)` in verify.md's own "UI
conformance" section — a deliberate calibration choice, not an
unexplained absence.

Eligible: check `.spine/capabilities.json` for `ui-conformance`'s
status in that repo. If `implemented`, run it directly — **not through
`floor`** — `.spine/adapters/ui-conformance` (CWD at that repo's
root, per `core/ADAPTER-CONTRACT.md §3.9`). Record pass/fail in its own
"UI conformance" section (§5) — never folded into "UI render," even
though both gate on the same `ui-touch` result; they check different
things (a real render vs. declared-token/component fidelity) and each
gets its own line so a reader can tell which one failed. If not
`implemented` (`unavailable`/`not-applicable` — the common case for a
project with no `docs/ui/` bundle at all), record it degraded with
its recorded reason.

## 1e. UI fidelity check (only when the diff touches a declared UI path)

The one check that compares a real render against the handoff's screenshots.
`ui-conformance` (1d) proves declared components exist; this proves the
screen *looks like its reference*, state by state, and that no copy or
number appeared that the spec and screenshots don't define
(`core/ADAPTER-CONTRACT.md` §3.10). Reuses step 1c's `ui-touch` result,
never a second invocation; `ui_touched` `false` for every repo checked:
skip this step entirely, no section (same omit-whole-section rule).

Class eligibility, its own key:

```
jq -r '.ui_fidelity_class1_optin // false' ~/.spine/user-config.json
```

Class 2: always eligible. Class 1: only if `true` (default `false`). Not
eligible: record `SKIPPED (Class 1, ui_fidelity_class1_optin not set)` in
the "UI fidelity" section — a deliberate calibration choice, worded as one.

Eligible: check `.spine/capabilities.json` for `ui-capture`. Not
`implemented` (`unavailable`/`not-applicable`): record it degraded with its
recorded reason in the section and under Capability gaps — never silently
skipped. `implemented`:

1. Run `.spine/adapters/ui-capture` directly (**not through `floor`**; CWD at
   the repo root) with `SPINE_UI_CAPTURE_DIR=<abs>/work/<task-id>/artifacts/ui-fidelity`.
   Non-zero exit is a real failure of this step (route won't render, browser
   error): record the adapter's diagnostics in the section and stop here —
   never turn a failed capture into "no findings." (A state recorded in
   `coverage.json` as `driver_failed` is not an adapter failure; the reviewer
   reports it.)
2. Dispatch one fresh `ui-fidelity` subagent (Agent tool) **per screen** in
   the capture dir, in parallel, each given only the capture dir path, the
   screen id and the task id — no plan, no diff, no implementation narrative.
   Budget per agent roughly 1 minute per captured state, at least 5 and at
   most 20 minutes; on overrun cancel it and record the screen as not
   reviewed. Before dispatching, use `core/scripts/adversary-cache-tier
   <task-id> ui-fidelity` with the screen's changed view/content files **plus
   its spec JSON and every screenshot it maps** as the blast radius, so a
   replaced mockup invalidates a prior clean pass.
3. Merge the per-screen replies into one JSON object
   (`agent: "ui-fidelity"`, concatenated `attacked` and `verdicts`), write it
   to `work/<task-id>/artifacts/ui-fidelity-raw.json`, and filter it:
   `${CLAUDE_SKILL_DIR}/../../scripts/verdict-filter <raw> --out
   work/<task-id>/artifacts/ui-fidelity-verdict.json --project <project root>`.
   Findings whose `render` evidence doesn't resolve are dropped there.

Findings **never change `/verify`'s PASS/FAIL** (a model's judgment is not a
floor gate); a failed capture does fail this step's line. The teeth are at
`/ship` §3a, which requires a human disposition for every kept, unfixed
finding.

## 2. Adversary count

Class 2: always both, `falsifier` and `security`. Class 1: how many adversaries
is a team default first, a personal default second — read
`<project root>/.spine/profile.json`'s `class1_adversaries` (1 or 2); if there
is no profile or the field is absent, fall back to `~/.spine/user-config.json`'s
`ceremony.class1_adversary_count`; if both are absent, default 2 (the team
profile wins where both exist — `core/ADAPTER-CONTRACT.md` §7's precedence rule).
1 means falsifier only — it's the one that proves the implementation against
its own plan, which is why it's never optional and the profile can never set it
to 0 (`profile-check` rejects that).

## 3. Run the adversaries

**Adversary re-run caching: content-aware, severity-aware, budget-tiered —
never a scope-narrowing shortcut** (`docs/tradeoffs.md`, "design-mode
adversary cost tiers" and its "content-hash correction" addendum). Before
dispatching `falsifier` (and `security`, if running), compute this task's
current blast radius for that agent — the diff's changed-file set (step 1's,
bookkeeping already excluded) union this run's fresh `callers.md` file
list, plus `dep-diff.md`'s for `security`, plus (multi-repo) any
`contract-touch.json` touched-contract spec paths for `falsifier`'s
cross-repo mandate — then get the tier mechanically, not by hand-deriving
it:

```
${CLAUDE_SKILL_DIR}/../../scripts/adversary-cache-tier <task-id> <agent> \
  --project <project root> <<'EOF'
<blast-radius file path>
<blast-radius file path>
...
EOF
```

Prints exactly one of three outcomes to stdout (reasoning to stderr):

- **REUSED** — every blast-radius file's current content hash matches
  `work/<task-id>/artifacts/<agent>-coverage.json` exactly (keyed on
  content hash, not path — a fix to a file already in the diff is exactly
  the new content no adversary has seen, so path-only matching would
  wrongly call that file "already covered"), *and* that prior pass's
  `<agent>-verdict.json` kept no `medium`/`high` finding (empty or
  `low`-only qualify — a literal empty-verdict requirement made this tier
  fire close to never, since a real pass over a nontrivial diff almost
  always surfaces at least one low/informational note). Don't dispatch;
  carry the existing `<agent>-verdict.json` forward, record `REUSED
  (content and blast radius unchanged, prior clean-enough verdict from
  <ran_at>)` in place of an Attacked/kept/dropped line (§5).
- **FOCUSED** — some but not all of the blast radius matches by hash, and
  the prior verdict kept no `high` finding. Dispatch with the **full**
  current diff — never narrowed, same non-negotiable rule as always — but
  name explicitly, in the delegation message, which files are
  unchanged-since-last-clean-pass (with that pass's timestamp), and reduce
  the wall-clock budget below the normal Class-based figure roughly in
  proportion to how much of the blast radius is unchanged (a judgment
  call, not a formula). The adversary keeps full authority to flag
  anything anywhere, including a cross-file interaction between changed
  and unchanged files — this tier narrows *effort allocation*, never
  *scope*.
- **FULL** — the catch-all: no coverage file yet (first dispatch), no
  blast-radius file matches by hash at all, or the prior verdict carried a
  `high` finding regardless of how much of the blast radius is otherwise
  stable (a round confirming a high-severity fix is never quietly
  downgraded to reduced effort). Dispatch fresh, full diff, full budget,
  same as a first pass.

Only *effort* is ever tiered, never *input* — every outcome except REUSED
hands the adversary the complete current diff, since the risk these agents
exist to catch is precisely in how a new change interacts with code that
didn't change (this is why per-finding or per-file input caching stays
rejected — see `docs/tradeoffs.md`).

Record coverage at dispatch time, not after: whenever an adversary *is*
dispatched (fully or focused), write `work/<task-id>/artifacts/
<agent>-coverage.json` — `{"files": {"<path>": "<sha256 of that file's
content as given to the adversary>", ...}, "contracts": [...], "ran_at":
"<ISO timestamp>"}` — alongside its verdict files, so the *next* `/verify`
pass on this task, or this run's own mid-verify re-check a few paragraphs
below, has something to compare against.

Each adversary is a fresh subagent (Agent tool) — give it only the plan
path, the diff (`git diff <base>..HEAD` — default base `HEAD`, i.e.
whatever's currently uncommitted, since implementation work isn't committed
until `/ship`; pass a different base if the human made interim commits), and
the artifact paths from step 1. Never summarize the implementation session
into the delegation message — that defeats the independence the fresh-
context property exists for (Layer 3).

**Resolve and state the base as a concrete SHA, not just symbolic `HEAD`.**
`falsifier` runs `isolation: worktree` (`core/agents/falsifier.md`), cloned
from this repo's own `HEAD` at dispatch time — a git ref, not this
session's uncommitted working tree — and its mandatory step 0 now applies
the diff into that worktree before anything else (`core/agents/
falsifier.md`, added ahead of mandate (a)). That apply only succeeds
cleanly when the worktree's own base matches the base the diff was
actually computed against. Run `git rev-parse <base>` yourself before
dispatch and include the resulting SHA verbatim in the delegation message
alongside the diff — this is what lets falsifier, on an apply failure,
report the precise mismatch (`git rev-parse HEAD` inside its own worktree
versus the SHA you gave it) instead of a vague "didn't apply." `security`
runs with no `isolation` (read-only, no worktree — `core/ADAPTER-CONTRACT.md`
§3.6) and already sees this session's real, uncommitted working tree
directly, so it has no analogous apply step and needs the base SHA only
for its own reference, not for reconciling a cloned copy.

**Bound each adversary's run — new, and load-bearing in `auto`.** An adversary
with no budget can run unbounded; in `guided` a watching human interrupts, but
`auto`/`checkpointed` have no such human, so the budget *is* the backstop. Give
each adversary a wall-clock budget proportional to blast radius, not to how
interesting it finds the code: roughly ~5 min for a Class 1 `auto` task, ~10 min
for Class 1 `checkpointed`/`guided`, ~20–30 min for Class 2. Size its scope to
match — a trivial Class 1 change gets a focused pass, not the full-ceremony sweep
a Class 2 warrants (this resolves the long-open "design-mode adversary cost
tiers" question for the normal path too, `docs/tradeoffs.md`). If an adversary
exceeds its budget, **stop it** (cancel the subagent) and record it in
`verify.md` as `adversary: <name> exceeded budget — not a clean pass` — never
silently treat a killed or timed-out adversary as PASS. In `guided`/
`checkpointed` that's a note for the human to act on; **in `auto` a budget breach
is an exception-stop** (`core/skills/task/SKILL.md` §5) — pull the human in
rather than shipping on an incomplete adversarial pass. Say it in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start short -->
> **What happened:** a reviewer (`<name>`) ran past its time limit and I stopped it.
> **What it means for you:** that review is incomplete, so I haven't counted it as a pass and won't ship on it.
> **To continue:** tell me to re-run just that review, or tell me how you'd like to handle it.
<!-- touchpoint:end -->

- `subagent_type: falsifier` — delegation message points at
  `work/<task-id>/plan.md`, the diff, and `work/<task-id>/artifacts/
  callers.md` (single-repo) or, multi-repo, each edited repo's own
  `<repo-abs-path>/work/<task-id>/artifacts/callers.md` (plural — one per
  edited repo, real paths, not a single merged file). **Single-repo: state
  the absolute project root path too**, not just the relative
  `work/<task-id>/...` paths — falsifier's own worktree clone generally has
  no `work/` folder at all (it's generated fresh this run, never
  committed), so it needs an unambiguous real path to resolve `callers.md`
  against directly, mirroring design mode's own `docs/charter.md` carve-out
  (`core/agents/falsifier.md` mandate (c)). Multi-repo already gives this —
  each `<repo-abs-path>/...` path is already absolute. **Multi-repo, when
  step 1b found any touched contract**: additionally include the
  workspace root's `work/<task-id>/artifacts/contract-touch.json`'s
  `touched_contracts` list and each one's spec file path — this is what
  arms `core/agents/falsifier.md`'s "Cross-repo mandate (d)." Say
  explicitly in the delegation message that this list is present so
  mandate (d) applies; falsifier never infers it from the diff alone.
  Also say explicitly whether `.spine/adapters/worktree-prep` exists and is
  `implemented` (read from `capabilities.json`) and its path — this is what
  arms mandate (b)'s step 0; falsifier never infers availability from the
  diff or from probing the filesystem itself. Also state the concrete base
  SHA resolved above, verbatim — this is what arms the diff-apply step 0
  ahead of mandate (a): falsifier compares it against its own worktree's
  `git rev-parse HEAD` to name a stale-base mismatch precisely if the
  apply fails, rather than guessing.
- `subagent_type: security` (if running) — same, plus each edited repo's
  own `work/<task-id>/artifacts/dep-diff.md`.

Each replies with exactly one JSON object (`core/ADAPTER-CONTRACT.md §5`).
Write each verbatim to `work/<task-id>/artifacts/<agent>-verdict-raw.json`,
then:

```
${CLAUDE_SKILL_DIR}/../../scripts/verdict-filter \
  work/<task-id>/artifacts/<agent>-verdict-raw.json \
  --out work/<task-id>/artifacts/<agent>-verdict.json
```

**Only ever read the filtered file when assembling `verify.md`.** The raw
file exists for audit, not for you to reason about — a verdict
`verdict-filter` dropped is not a verdict, regardless of how it reads.

**Collect every adversary's verdict before applying any fix — never mutate the
tree while an adversary is still running.** Each adversary reasons about the
snapshot it was dispatched against; fix a finding (edit the tree) while another
adversary is still running and that one is now verifying stale code — it will
re-report a defect you've already fixed, burning its whole budget on it. This is
real, not hypothetical: it's the exact waste the `auto` fire-test surfaced (a
falsifier spending 30 min rigorously confirming a lint error the main session had
already fixed mid-verify). So: dispatch all adversaries, **wait for every one to
return** (or hit its budget), filter the verdicts, and only *then* apply fixes.
If the fixes are substantive (more than a comment or rename), recompute
`callers.md`/`dep-diff.md` against the fixed tree and re-run
`adversary-cache-tier` the same way as step 3's top. A fix changes the
content of whatever file it touches, so that file's hash no longer matches
`<agent>-coverage.json` by construction — **REUSED never applies to a file
you just fixed**, regardless of whether its path was already sitting in
the diff; fixing a bug is exactly the new content an adversary needs to
see. Same three outcomes apply, mechanically, from the recomputed blast
radius and the carried-forward verdict's severity. (Worktree note: the falsifier runs the project's `worktree-prep` adapter, if
one is `implemented`, as step 0 of its stub-out probe — a single bounded
declared command to make its isolated worktree's toolchain resolvable
(`core/ADAPTER-CONTRACT.md §3.6`). When no such adapter is implemented, the
stub-out probe degrades to a recorded tooling-gap verdict rather than the
falsifier burning budget hand-bootstrapping — surfaced in `verify.md` per the
"never silently skip a gate" discipline, same as any other degraded
capability.)

## 3.5. Secret scan on adversary evidence

Adversary verdict evidence can carry real captured command output
(`core/ADAPTER-CONTRACT.md §5`'s `{"kind":"command","output":"..."}`
evidence shape) — a debugging command constructed to prove a scenario can
print an env var, a connection string, or a token, and that output is
written verbatim into `work/<task-id>/artifacts/<agent>-verdict-raw.json`
and `<agent>-verdict.json`, both committed as part of the task folder.
Nothing else catches this: step 1's floor (including `secret-scan`) runs
*before* the adversaries produce these files, scoped to the diff, and
never sees them. This step exists because without it, nothing ever
re-scans after they're written.

For every verdict file step 3 just wrote this pass (raw and filtered, only
the agents that actually ran or actually re-wrote a file this pass — a
`REUSED` verdict wrote nothing new and has nothing new to scan here):

```
printf '%s\n' work/<task-id>/artifacts/<agent>-verdict-raw.json \
  work/<task-id>/artifacts/<agent>-verdict.json \
  | .spine/adapters/secret-scan
```

**Exit 0**: clean, record it as a one-line pass in verify.md's "Adversary
evidence secret scan" section, continue. **Non-zero**: a real, serious
finding — quote `secret-scan`'s own diagnostics verbatim in that section,
and say plainly, in the same place: **redacting the flagged file is not
enough — a captured secret was, by definition, real at the moment it was
captured, and only the person who owns that credential can rotate it;
spine can flag the exposure, it cannot undo it.** This is a `/verify` FAIL
reason (step 6), distinct from every other one — report it exactly as
"secret detected in adversary evidence" so `/task`'s FAIL handling routes
back to implementation the same as any other failure, and the human sees
the rotation instruction, not a generic fail.

If `secret-scan` is not `implemented` in `.spine/capabilities.json`
(unusual on any project that ran `/design` — its §2 requires this
capability `implemented`, never left at a placeholder status), record it
in verify.md's "Capability gaps" section like any other degraded
capability rather than blocking — this step degrades the same way every
other capability-gated check in this file already does.

## 4. Conformance

Single-repo:

```
${CLAUDE_SKILL_DIR}/../../scripts/conformance work/<task-id>/plan.md \
  --out work/<task-id>/artifacts/conformance.json
```

**Multi-repo**: `conformance` itself is unmodified — it still takes one
`--project` and one plan file. Run it once per edited repo: write a scratch
copy of `plan.md`'s `## Predicted touch` section containing only that
repo's entries with the `<repo-name>:` prefix stripped
(`work/<task-id>/artifacts/predicted-touch-<repo>.md`, a minimal file with
just the `## Predicted touch` heading and that repo's own bullets — this
is what `conformance`'s own parser needs, nothing more), then
`conformance work/<task-id>/artifacts/predicted-touch-<repo>.md --project
<repo-abs-path> --out work/<task-id>/artifacts/conformance-<repo>.json`.

Never blocks anything (Layer 4) — it scores the plan, not the change.

## 5. Assemble verify.md

Fill `${CLAUDE_SKILL_DIR}/../../templates/verify.md`'s structure into
`work/<task-id>/verify.md`, sourced only from the files written above —
this document quotes scripts, it doesn't paraphrase them:

- Floor results table from `floor-result.json`'s `results` array — one row
  per entry, pass/fail/degraded verbatim. Multi-repo: one whole "Floor
  results — `<repo-name>`" section per edited repo, from that repo's own
  `floor-result-<repo>.json` — never merge two repos' rows into one table.
- **Contract conformance** (multi-repo only, per `core/templates/
  verify.md`'s own section): one line per contract `contract-touch`
  reported touched — name, `spec_change`, `registry_stale` — and, for every
  gated-not-edited consumer from step 1b, its `contract-check` result.
  "None" only if `contract-touch` found nothing touched.
- **UI render** (per `core/templates/verify.md`'s own section, omitted
  entirely if step 1c found no UI path touched in any repo): the
  `ui-render` result — pass/fail with the adapter's own diagnostics on
  fail, `SKIPPED (Class 1, ui_render_class1_optin not set)` if this class
  wasn't eligible, or degraded with its recorded reason if the capability
  isn't `implemented`.
- **UI conformance** (per `core/templates/verify.md`'s own section,
  omitted entirely under the same condition as "UI render" above — they
  share one `ui-touch` result): the `ui-conformance` result — pass/fail
  with the adapter's own diagnostics on fail (which declared component or
  token didn't show up in the real render), `SKIPPED (Class 1,
  ui_conformance_class1_optin not set)` if this class wasn't eligible,
  or degraded with its recorded reason if the capability isn't
  `implemented` (the common case for a project with no `docs/ui/`
  bundle).
- **UI fidelity** (per `core/templates/verify.md`'s own section, omitted
  entirely under the same condition as "UI render"): the capture result,
  then per screen the states compared vs not compared (with
  `coverage.json`'s reason), then each kept `ui-fidelity` verdict —
  severity, state, claim, and its `render` evidence — and kept/dropped
  counts from `verdict-filter`. Or `SKIPPED (Class 1, ui_fidelity_class1_optin
  not set)`, or degraded with `ui-capture`'s recorded reason. Write the same
  `disposition` (`fixed`/`not_fixed`) back into `ui-fidelity-verdict.json` as
  for the adversaries. `ui-fidelity` is not an adversary for §2's
  `class1_adversaries` count.
- Conformance line from `conformance.json` (or one line per
  `conformance-<repo>.json`, multi-repo).
- One subsection per adversary that ran *or was reused*, from its filtered
  verdict file: the `attacked` list, then each kept verdict, then the
  kept/dropped counts (dropped count comes from `verdict-filter`'s own
  stderr line, capture it when you run step 3) — or, for a reused verdict,
  `REUSED (blast radius unchanged, prior clean verdict from <ran_at>)` in
  place of that line, per step 3's skip condition. **For every kept verdict,
  as you decide how to word its bullet, also write that decision back**
  (`core/ADAPTER-CONTRACT.md §5`'s `disposition` field) into
  `work/<task-id>/artifacts/<agent>-verdict.json`: `"fixed"` if this task's
  own diff resolved it (whether before this assembly or in an earlier round
  superseded by this one), `"not_fixed"` for anything deliberately left —
  out of scope, a design question for later, or genuinely still open. This
  is the same call already being made in the bullet's own prose ("—
  fixed.", "— flagged, not fixed.", etc.) — one more field recording it
  mechanically, not a second judgment. Leaving the field off a verdict is
  never equivalent to marking it fixed; `/ship`'s flagged-finding triage
  (`core/skills/ship/SKILL.md` §3a) treats an absent field as `not_fixed`
  specifically so a verdict this step forgot to mark never silently reads
  as resolved.
- **Adversary evidence secret scan** (step 3.5): one line, pass or fail
  with `secret-scan`'s own diagnostics quoted verbatim on fail, plus the
  redact-is-not-enough/rotate-the-credential note from step 3.5 — never
  omit that note on a fail, it's the whole reason this section exists.
- Capability gaps: every capability in `.spine/capabilities.json` marked
  `unavailable`/`not-applicable` that this class would otherwise have run
  (Class 2 also implies `mutate` and, per the migration lane,
  `migrate-rehearse` if this task touched a migration path), with its
  recorded reason. Empty only if nothing degraded.
- **Tooling gaps** (distinct from capability gaps above — this is about
  spine's own scripts being unreachable, not a project capability being
  unimplemented): merge every `TOOLING GAP:` line already accumulated in
  `work/<task-id>/notes.md` across earlier phases with anything hit during
  this `/verify` run itself. One line per gap: script name, and the
  consequence exactly as recorded. Write "none" only if genuinely empty —
  this section exists specifically so it's never silently absent when it
  shouldn't be.
- **Setup events** (distinct from all three sections above, and from
  `deviations.md`: this project's own tooling config —
  `.spine/adapters/*`, `.spine/capabilities.json`,
  `.spine/protected-paths.conf` — needing a fix for a reason unrelated to
  whether the plan's understanding of *the product* held up, per
  `core/skills/task/SKILL.md` §4's boundary test): merge every `SETUP:`
  line already accumulated in `work/<task-id>/notes.md`, verbatim, one
  line per event. Write "none" only if genuinely empty.


## 6. Report

Reply to the caller with exactly: `PASS` or `FAIL`, plus the one-line reason
if FAIL (which capability, which repo's floor, an ungated consumer's failed
`contract-check`, step 1b's breaking-change gate, step 1c's `ui-render`
failure, step 1e's `ui-capture` failure, or step 3.5's "secret detected in adversary evidence" — "adversaries
ran, see verify.md" is not a FAIL by itself — adversary *findings* don't
fail verify, they inform `/ship` and the briefing; a secret in the
adversary's own *evidence* is a different thing entirely and does fail
verify). `/task` decides what happens next; this skill's job ends at
`verify.md` plus that one-line verdict. When a human ran `/verify` directly,
also leave them this report form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start report -->
> **Bottom line:** <checks passed | checks failed>, in one plain sentence, and whether anything waits on you.
> **What I did:** <2-3 short lines: what was checked and by whom, in plain words>
> **What you need to do:** <`/ship <task-id>` | fix <the one thing> and re-run>
> **Worth knowing:** <problems found but not blocking, and when each would matter, one line each; or "nothing">. Details: `work/<task-id>/verify.md`.
<!-- touchpoint:end -->
