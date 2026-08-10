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

Report that line first, verbatim, before anything else. Then:

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

Don't editorialize with targets or thresholds this skill doesn't have —
Layer 1 calibration (build prompt §0) intentionally sets no budget cap by
default and expects `/costs` data, not guesses, to justify one later. If the
untracked ratio or bypass count looks high, say so plainly and let the
engineer decide what it means; this command's job is to surface numbers
the engineer would otherwise have to dig for, not to interpret them for
them.
