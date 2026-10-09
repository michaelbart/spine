# Tradeoffs

Honest costs and limits. `README.md` says what spine is for; this says what it costs,
where it is the wrong tool, and what it does not catch. Numbers come from
`core/scripts/spine-stats` run on turnpilot and tgml (`docs/baseline-2026-10.md`,
2026-10-08); re-run it before trusting them. Design rationale for individual features
lives in `docs/design/`.

## What it costs

**Tokens.** In turnpilot, spine skills and their subagents account for about 58% of
fresh input tokens and 30% of output; the rest is implementation plus any non-spine
conversation, and the two cannot be separated. A task takes a median of about 1.0M
fresh-input-plus-output tokens and 2.0 hours of active time (gaps over 30 minutes
excluded). The largest single items are `/ship` on the main thread, the falsifier
adversary and `/task`. There is no spine-free comparison, so this says what spine
costs, not whether it pays for itself.

**Human minutes.** Not measured yet. Class 1 asks for three short touchpoints
(confirm the class, read the plan before approving it, read the briefing) and Class 2
adds more stops and a higher chance of a real deviation; the earlier estimates were
a few minutes each for Class 1 and 10 to 20 minutes for Class 2. A `/design` session
is closer to a milestone-sized review (20 to 40 minutes). Treat these as guesses.

**Hook friction.** Before the 2026-10-08 parser fix, turnpilot saw about 27 hook
stops per shipped task. Nearly all were Bash commands the write-target parser could not
place: a relative path after `cd` (the largest group), a variable in the target, or a
quoted `>` taken for a redirect (about one in ten). The parser now resolves a literal
`cd x && ...` chain and ignores quoted text; a variable target still stops. Re-measure
with `spine-stats friction` after a few weeks of use.

**`/prototype`** is cheap by design: no plan, no floor, no adversary. If a prototype
session regularly runs long it has become implementation and belongs in a `/task`.

**`/wayfinder`** spreads its cost across many sessions. Watch a map that spawns
tickets faster than it resolves them: that effort was never one map's worth of scope.

## Where this is the wrong tool

- **Tiny repos you hold in your head.** The artifact trail exists to compensate for
  context lost across sessions. A script you wrote ten minutes ago has no such problem.
- **Genuinely novel design work.** Research assumes there is an existing "how it works
  today" to ground in. Net-new architecture has none, so research would come back
  empty or invent false grounding.

## When to abandon it

If the attention cost of the touchpoints regularly exceeds what a careful human review
of the same diff would cost, after calibration and after `/ratchet` has turned repeat
friction into a deterministic check, the system has failed its own test.

## What makes this obsolete

The artifact trail exists because a model session is a stateless function of its
context window. A model that reliably carries long-horizon state across sessions turns
the ceremony around those artifacts into pure cost.

## Measured tradeoffs

- **The gates trust files the model writes.** `docs/enforcement-map.md` classifies 92
  rules: 53 are prose only, 26 depend on a script that a skill must tell the model to
  run, and 11 are hooks (4 of those are the Claude Code harness's own frontmatter
  rules). `phase-gate` and `path-escalate` read the `state` and `class` files, and a
  task folder is writable during research and plan, so a model can move itself out of a
  gated phase. `set-state` now refuses the common accidental skips (implementing with no
  approval, shipping without a passing verify or with open deviations), but it reads files
  the model also writes. The system assumes a cooperative model and catches mistakes;
  it is not a defence against one that is trying to get around it.

- **Adversary findings inform `/ship`; they do not gate it.** A `high` finding does not
  fail `/verify`. At ship time it prompts (`guided`) or goes to the milestone's Known
  gaps (`auto`). The hard gates are the deterministic floor, zero open `deviations.md`
  entries, and the render checks. Deliberate: a hard block on adversary opinion recreates
  the review-bottleneck rubber-stamping this system exists to avoid. In turnpilot about
  half of what the falsifier and security adversaries raise ends up fixed (53% and 50%;
  26% and 24% in tgml), so the review is finding things; but it is disclosure, not
  enforcement.
- **Plan quality has no backstop.** The falsifier tests an implementation against the
  plan's own acceptance checks; nothing checks that those were the right checks. The load
  sits entirely on whoever approves the plan, and an `auto` task skips that approval, so
  the falsifier's stub-out probe checks a plan nobody reviewed. A lint on the shape of
  acceptance checks was considered and declined: it would only catch empty plans.
- **The deviation circuit breaker runs on an honor system.** It counts records actually
  written to `deviations.md`. Two backstops now exist: `/verify` adds an "unlogged
  deviation" line when the diff touches files the plan did not predict and no deviation
  was recorded, and each logged deviation is a `deviation` event. Misclassifying a real
  deviation as a *setup event* (a project-config problem, not counted) is still possible.
- **Class 0 has a lint and type check, nothing more.** A change under the trivial threshold
  runs `floor 0` (types and lint on the changed files) and logs the result. An off-by-one
  that type-checks and lints still ships on the model's own judgment; `path-escalate`
  catches a protected-path touch after the fact.
- **Class 2 needs a smoke-testable runtime, unless the charter says there is none.**
  `smoke-seed/run/golden` have no general `not-applicable` escape hatch at Class 2.
  A project can declare `Runtime: none` in its charter, mark `smoke-run` not-applicable
  for "no runtime", and have `test` implemented; then the gate is waived as DEGRADED
  (never a pass), by the same three-independent-facts rule as the M0 waiver.
- **Some floor layers have never run.** `callers`, `clone-scan` and `mutate` are
  unavailable in every recorded task in both projects, so the promise that the floor
  fails on new duplication is not true there until an adapter exists. `/spine` reports
  checks skipped across many verifies.
- **Write-blocking hooks do not see every way to write a file.** They recognise
  `Edit`/`Write` and common Bash shapes and fail closed on a target they cannot place.
  Text that could be executed (`eval`, `source`, `awk`, `xargs`, `find`, or inline code
  given to a shell or interpreter such as `bash -c`, `python3 -c`, `node -e`, a heredoc
  fed to one, or a pipe into a shell) is scanned in full, quotes and heredocs included. A write buried
  inside an interpreter's own call (`python -c "open(...)"`) is still invisible.
- **Adapters are trusted between recalibrations.** `adapter-conformance` is a
  black-box exit-code and output-shape check: it cannot tell a fake-but-passing self-test
  from a real one, and the `SELF-TEST-FAIL-FIXTURE-BROKEN` marker that catches one
  narrow case is opt-in per adapter.
- **Adversary evidence is checked for shape, not truth.** A finding needs a real
  `file:line` or command output to survive filtering; nothing confirms the evidence
  supports the claim.

## Inherent limits (no fix planned)

- A per-task floor sees only the current diff, so debt in untouched files stays invisible.
  For a whole-tree check run `floor 1 --full` in CI (nothing runs it automatically).
- Two tasks that touch logically related but textually disjoint code can pass their own
  verification and still combine badly.
- A charter section that never collides with reality can rot undetected.
- Seeded smoke data covers only the shapes someone thought to seed.
- Concurrency defects are out of scope; there is no stress lane.
- Secrets, credentials and PII have no auto-loaded discipline rule; `secret-scan` and the
  project's protected paths are the only coverage.
- `briefing.md` heading conventions bind only tasks shipped after they changed.
- Two tasks in one checkout are not coordinated: there is a single `.spine/current-task`.

## Experimental: `/autopilot`

Removes every human stop `/task` has, including the ones spine treats as structural
(Class 2's guided stops, halt-tier deviations, the circuit breaker), and defers all
review to one end-of-run report. It removes stops, never checks: the floor, the
falsifier's stub-out probe, the security adversary and `conformance` stay real. Every
commit stays local; it never pushes or opens a PR. Used twice in turnpilot; not the
default or recommended way to work.

## Deferred (not built)

| Not built | Note |
|---|---|
| Parallel or team-of-agents orchestration | Every flow is strictly sequential: one active task, one active map. |
| Cross-model adversary routing | Every agent inherits the session's model. Worth a pilot once there is enough adversary-yield data to see whether it helps. |
| Scheduled cleanup of pre-existing duplication | Duplication checks run only against a task's changed files. |
| A product-spec layer | The charter stays at constraints; a product spec stays optional and human-authored. |
| A concurrency or stress-test lane | The smoke capability is the insertion point. |
| Reverting a shipped task | `git revert` is adequate solo; a reviewed compensating task would be a design of its own. |
