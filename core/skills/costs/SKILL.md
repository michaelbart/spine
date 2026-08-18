---
name: costs
description: Report cost instrumentation — untracked-commit ratio first, then per-task ledger aggregates.
disable-model-invocation: true
argument-hint: [--since <git-date>]
---

You are running `/costs`. Parse an optional `--since <git-date>` from
`$ARGUMENTS` (default: no lower bound — full history).

**Surface the untracked-commit ratio first, never buried** (build prompt
§2.6) — this is the deliberate close for the blind spot Class 0 tasks leave:
they write no ledger entry, so drift toward doing everything as Class 0
would otherwise look like an absence of data, indistinguishable from a
quiet week.

```
${CLAUDE_SKILL_DIR}/../../scripts/ledger scan-untracked-ratio [--since <date>]
```

Report that line first, verbatim, before anything else. **Immediately follow
it with the same caveat `core/scripts/render-dashboard` already carries in
its own instrument-note** (don't let this exist in only one of the two
surfaces): only the final `/ship` commit for a task carries a `Spine-Task:`
trailer (`core/ADAPTER-CONTRACT.md` §6) — every intermediate task-bookkeeping
commit the `task` skill's own `registry-sync` calls produce (`task: open`,
`task: plan approved`, `task: verify complete, PASS`, `task: done`, etc.)
legitimately carries none of the three trailers and is *not* off-spine work.
On a project running every task through spine, expect this ratio to look
high by construction — dominated by spine's own lifecycle commits, not by
real gaps. Say this before the number invites the wrong conclusion, not
after: a bare ratio reads as an indictment; this caveat is what keeps it an
instrument. It does not excuse a genuinely high ratio on a project with real
off-spine work mixed in — the caveat explains the mechanical *source* of
inflation, it doesn't zero it out; if bypass count or tooling gaps are also
nonzero, or the project has commits with no task association at all, still
say so plainly. Then:

```
${CLAUDE_SKILL_DIR}/../../scripts/ledger aggregate [--since <date>]
```

Report `task_count`, `total_tokens`, `avg_deviation_count`,
`class_escalation_count`, `bypass_count`, `tooling_gap_count`, and
`hand_tracked_task_count` from its JSON. Put the bypass count next to the
untracked ratio in your summary, not at the bottom — both are "work that
happened outside the normal gates," and the build prompt is explicit that
bypass must stay visible, never quiet.

**`tooling_gap_count` and `hand_tracked_task_count` go right alongside
them, not at the bottom either.** These count a different failure mode
than bypass — not "the engineer chose to skip a gate" but "the tooling the
engineer thinks is running silently wasn't." `tooling_gap_count` is the
total number of individual could-not-run events across all tasks (`ledger`,
`check-stale`, `conformance`, `verdict-filter`, or `floor` itself unreachable
for some task); `hand_tracked_task_count` is the narrower, more serious
count of tasks where `ledger` itself was unreachable and the whole task's
ledger.json had to be hand-authored rather than script-produced. A rising
`tooling_gap_count` means the engineer is running a lighter system than
they think they are — say so plainly if it's nonzero, the same way you
would for a rising bypass count. Zero here is not proof nothing degraded —
it only counts gaps that got recorded; see `docs/tradeoffs.md`'s Auto Mode
classifier wall section for the residual case where even the recording
mechanism (`ledger`) was the thing that failed.

**Team profile & drift (Phase 5).** Report the active profile so the numbers
have context: read `<project root>/.spine/profile.json` (absent = built-in
`standard`) and name it plus `class1_adversaries` / `autonomy_ceiling` — a team
running `prototype` (one adversary, `auto` allowed) is *expected* to look
lighter than one running `regulated`, and comparing the two without saying so
misleads. Then two drift numbers, framed like the untracked ratio (surface, not
judge):

- **Traced-trivial count** — `wc -l < <project root>/.spine/trace.jsonl` (0 if
  absent): how many Class 0 changes were traced this project's life. A high and
  fast-rising count next to few real tasks is the same "sliding toward Class 0"
  signal the untracked ratio catches one level up — worth naming, not alarming.
- **Below-recommendation downgrades** — count `work/*/ledger.json` with a
  `class_downgraded_from` field set (engineers who took `/intake`'s menu below
  the recommended class). A few are normal judgment; a pattern in one area
  means either the recommender is miscalibrated or rigor is being dodged — say
  which the data can't tell you, and let the team decide.

**Per-engineer view (Extension C §2.7), when more than one distinct
`engineer` appears in the aggregate above:**

```
${CLAUDE_SKILL_DIR}/../../scripts/ledger aggregate --by-engineer [--since <date>]
```

Report it as a table, one row per engineer. **Frame this as an
instrument, not a leaderboard** — say so explicitly if presenting it —
its only purpose is surfacing drift early (one person's work sliding
toward Class 0, bypass, or repeated `claims_conflicts`), the same way the
untracked-commit ratio surfaces it in aggregate. A metric read as a
leaderboard gets gamed into uselessness; don't rank engineers against each
other in your own summary, report the numbers and let the team decide what
they mean. `claims_conflicts` here is `core/scripts/claims-check`'s own
blocking-conflict count for that engineer's tasks — a rising count across
one person's tasks specifically (not the team total, already visible
above) is worth naming plainly, same standard as a rising bypass count.
Skip this whole subsection on a solo project (one engineer, or every
`ledger.json` predates Extension C and carries `engineer: null`) — nothing
to compare.

Don't editorialize with targets or thresholds this skill doesn't have —
Layer 1 calibration (build prompt §0) intentionally sets no budget cap by
default and expects `/costs` data, not guesses, to justify one later. If the
untracked ratio or bypass count looks high, say so plainly and let the
engineer decide what it means; this command's job is to surface numbers
the engineer would otherwise have to dig for, not to interpret them for
them.
