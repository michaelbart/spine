---
name: verify
description: Run the deterministic floor and the adversary agents for a task, and assemble verify.md. Invoked by /task at the verify phase — not a general code-review command.
disable-model-invocation: true
argument-hint: <task-id>
---

You are running `/verify` for task `$ARGUMENTS` (the task ID). Scripts live
at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`, the template at
`${CLAUDE_SKILL_DIR}/../../templates/verify.md`. Read `work/<task-id>/class`
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
only, never a floor (build prompt §2, edited-vs-affected rule — a
consumer's own pre-existing, unrelated failures must never block a
producer task forever).

**Tooling-gap discipline applies to every script below** — see
`core/skills/task/SKILL.md`'s own header section for the full three-outcome
rule (ran-passed / ran-failed / could-not-run) and how to record it
(notes.md, `ledger note-gap`, and a hand-authored `ledger.json` stub if
`ledger` itself is what's unreachable). Several capabilities here need a
specific default when they could not run at all, because "silently treat
as pass" is exactly the failure mode this fix exists to close:

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
  unmeasured for this task" — and leave `ledger set ... conformance_score`
  unset (don't invent a number).
- **`ui-touch` could not run at all**: do not treat this as "no UI
  touched" (that would silently skip step 1c's gate on a task that
  genuinely touched UI code) — record the gap with the consequence "UI
  render check eligibility unmeasured for this task; step 1c skipped
  without proof this diff didn't need it" and skip step 1c, loudly, rather
  than silently defaulting to not-touched.

## 1. Floor

Single-repo:

```
${CLAUDE_SKILL_DIR}/../../scripts/floor <class> --task <task-id> \
  --out work/<task-id>/artifacts/floor-result.json
```

**Multi-repo**: run this once per repo in the *edited* set, each with
`--project <repo-abs-path>` (from `workspace.json`) so it runs against that
repo's own `.spine/adapters/` and its own git history — `floor` itself is
unmodified (build prompt §2's edited-vs-affected rule lives in `/verify`'s
orchestration, not in `floor`'s own dispatch). `--task <task-id>` makes
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
  elsewhere. This is the deterministic check build prompt §2 requires; it
  reads the real diff's classification, never the plan's self-report alone.
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

**Skip an adversary entirely when its last verdict from this task is still
clean and its recorded coverage is a superset of the current blast
radius — this is the only sound form of adversary re-run caching**
(`docs/tradeoffs.md`, "design-mode adversary cost tiers"). Before
dispatching `falsifier` (and `security`, if running), check for a prior
`work/<task-id>/artifacts/<agent>-coverage.json` and `<agent>-verdict.json`
left by an earlier `/verify` pass on this same task — whether that was a
fresh invocation after a `FAIL`-and-fix cycle, or this run's own mid-verify
re-check below:

- **Clean and covered → skip.** `<agent>-verdict.json`'s `verdicts` array
  is empty *and* the current blast radius — the diff's changed-file set
  (step 1's, bookkeeping already excluded) union this run's fresh
  `callers.md` file list, plus `dep-diff.md`'s for `security`, plus
  (multi-repo) any `contract-touch.json` touched-contract names for
  `falsifier`'s cross-repo mandate — is a **subset** of
  `<agent>-coverage.json`'s recorded set. Don't dispatch that adversary;
  carry its existing `<agent>-verdict.json` forward unchanged, and record
  in `verify.md` (§5) `REUSED (blast radius unchanged, prior clean verdict
  from <its ran_at timestamp>)` in place of an Attacked/kept/dropped line.
- **Anything else → full re-run, never a narrowed one.** A prior finding
  (verdicts non-empty), a missing coverage file, or a blast radius that
  grew even partially all mean: dispatch the adversary fresh, with the
  *full* current diff and *full* current `callers.md`/`dep-diff.md` — same
  as a first run. There is no sound middle tier that re-checks only the
  new files; the risk these agents exist to catch is precisely in how a
  new change interacts with code that didn't change, so a partial pass
  can't be trusted to see that interaction (this is why per-finding or
  per-file caching was rejected, not just whole-adversary caching accepted
  — see `docs/tradeoffs.md`).

Record coverage at dispatch time, not after: whenever an adversary *is*
dispatched (below), write `work/<task-id>/artifacts/<agent>-coverage.json`
as `{"files": [...the file set actually given to it, sorted, deduped...],
"contracts": [...touched-contract names if any...], "ran_at": "<ISO
timestamp>"}` alongside its verdict files, so the *next* `/verify` pass on
this task — or this run's own mid-verify re-check a few paragraphs below —
has something to compare against.

Each adversary is a fresh subagent (Agent tool) — give it only the plan
path, the diff (`git diff <base>..HEAD` — default base `HEAD`, i.e.
whatever's currently uncommitted, since implementation work isn't committed
until `/ship`; pass a different base if the human made interim commits), and
the artifact paths from step 1. Never summarize the implementation session
into the delegation message — that defeats the independence the fresh-
context property exists for (build prompt §2.5 Layer 3).

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
rather than shipping on an incomplete adversarial pass.

- `subagent_type: falsifier` — delegation message points at
  `work/<task-id>/plan.md`, the diff, and `work/<task-id>/artifacts/
  callers.md` (single-repo) or, multi-repo, each edited repo's own
  `<repo-abs-path>/work/<task-id>/artifacts/callers.md` (plural — one per
  edited repo, real paths, not a single merged file). **Multi-repo, when
  step 1b found any touched contract**: additionally include the
  workspace root's `work/<task-id>/artifacts/contract-touch.json`'s
  `touched_contracts` list and each one's spec file path — this is what
  arms `core/agents/falsifier.md`'s "Cross-repo mandate (d)." Say
  explicitly in the delegation message that this list is present so
  mandate (d) applies; falsifier never infers it from the diff alone.
- `subagent_type: security` (if running) — same, plus each edited repo's
  own `work/<task-id>/artifacts/dep-diff.md`.

Harvest each subagent's own transcript in full into the ledger under its
own phase key, same mechanism as `core/skills/task/SKILL.md` §6:
`ledger harvest <task-id> falsifier --transcript
.../subagents/agent-<id>.jsonl --from <epoch> --to <now>` (and the same for
`security` if it ran), `<id>` from the Agent tool's result. This is a
separate line item from the `verify` phase key `/task` marks for this
skill's own orchestration overhead — don't fold one into the other.

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
`callers.md`/`dep-diff.md` against the fixed tree and apply the same
skip-if-subset check from step 3's top: your fix is itself a change no
adversary has seen, but if the fix's own blast radius stayed inside what
that adversary already covered, its clean verdict on everything *else*
still holds and only needs recording as `REUSED`, not a full second
dispatch — if the fix's blast radius grew at all (e.g. it touched a file
outside the original diff to satisfy an invariant), re-run that adversary
fully, same "no narrowed pass" rule as above. (Worktree note: an adversary needs its toolchain resolvable in its isolated
worktree — if the floor's own tools aren't reachable there, the adversary wastes
budget fighting tooling instead of the code; that's an install/agent-setup
concern to watch, tracked in `docs/tradeoffs.md`.)

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

Never blocks anything (build prompt §2.5 Layer 4) — it scores the plan, not
the change.

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
- Conformance line from `conformance.json` (or one line per
  `conformance-<repo>.json`, multi-repo).
- One subsection per adversary that ran *or was reused*, from its filtered
  verdict file: the `attacked` list, then each kept verdict, then the
  kept/dropped counts (dropped count comes from `verdict-filter`'s own
  stderr line, capture it when you run step 3) — or, for a reused verdict,
  `REUSED (blast radius unchanged, prior clean verdict from <ran_at>)` in
  place of that line, per step 3's skip condition.
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

`ledger set <task-id> conformance_score <f1 from conformance.json>` and
`ledger set <task-id> capability_gaps <json array of the gap names>`.

## 6. Report

Reply to the caller with exactly: `PASS` or `FAIL`, plus the one-line reason
if FAIL (which capability, which repo's floor, an ungated consumer's failed
`contract-check`, step 1b's breaking-change gate, or step 1c's `ui-render`
failure — "adversaries ran, see verify.md" is not a FAIL by itself —
adversary findings don't fail verify; they inform `/ship` and the
briefing). `/task` decides what happens next; this skill's job ends at
`verify.md` plus that one-line verdict.
