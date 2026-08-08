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
`class_escalation_count`, `bypass_count` from its JSON. Put the bypass count
next to the untracked ratio in your summary, not at the bottom — both are
"work that happened outside the normal gates," and the build prompt is
explicit that bypass must stay visible, never quiet.

Don't editorialize with targets or thresholds this skill doesn't have —
Layer 1 calibration (build prompt §0) intentionally sets no budget cap by
default and expects `/costs` data, not guesses, to justify one later. If the
untracked ratio or bypass count looks high, say so plainly and let the
engineer decide what it means; this command's job is to surface numbers
the engineer would otherwise have to dig for, not to interpret them for
them.
