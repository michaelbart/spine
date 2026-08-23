---
name: task-report
description: Generate a self-contained HTML visualization of one task's work/ record — timeline, deviations, adversary activity, floor results. Read-only, never a gate.
disable-model-invocation: true
argument-hint: <task-id> [--project <path>]
---

You are running `/task-report`. `$ARGUMENTS` is `<task-id>` plus an
optional `--project <path>` (default: current working directory). If the
task id is missing, ask for one before doing anything else.

Confirm `work/<task-id>/` exists under the resolved project root — if not,
say so and stop rather than letting the script fail with a less legible
message.

Hand this to the shell verbatim, `../../` included — do **not** lexically
collapse it to `.claude/`; `.claude/skills/task-report` is a symlink into
the spine core checkout.

```
${CLAUDE_SKILL_DIR}/../../scripts/render-task --task <task-id> --project <project root>
```

This reads whatever already exists in that task's folder — `ledger.json`,
`plan.md`, `deviations.md`, `verify.md`, `artifacts/*-verdict.json`,
`briefing.md` — and writes `work/<task-id>/report.html`. A task that
hasn't reached a given phase yet (no `verify.md`, no `briefing.md`) is
rendered with that section explicitly labeled absent, never fabricated or
left blank without explanation — this is read-only rendering, not a new
source of truth.

Report the output path back to the human plainly: `wrote
work/<task-id>/report.html — open it in a browser`. **Never a gate** —
this skill is not invoked by `/task`, `/verify`, or `/ship`, and nothing
downstream reads the generated file back in. If `render-task` itself
could not run (missing `jq`, unreadable input), say so directly; don't
retry silently or fall back to reconstructing the report by hand.
