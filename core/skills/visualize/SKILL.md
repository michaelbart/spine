---
name: visualize
description: Generate a project-wide HTML dashboard — task Gantt timeline, event feed, decision store, capability matrix, milestone progress, cost/drift instrument. Read-only, never a gate.
argument-hint: [--project <path>]
---

You are running `/visualize`. `$ARGUMENTS` is an optional
`--project <path>` (default: current working directory).

Confirm the resolved project root is spine-installed
(`.spine/capabilities.json` exists) — if not, say so and stop rather than
letting the script fail with a less legible message.

```
${CLAUDE_SKILL_DIR}/../../scripts/render-dashboard --project <project root>
```

This aggregates everything spine has already produced for the whole
project — every `work/<task-id>/` folder's class/state/verify result and
ledger phase timestamps (for the Gantt strip), the real git log (for the
event feed, flagging any `Spine-Bypass:` commit), `docs/decisions/D-*.md`
(with supersession chains), the live `.spine/capabilities.json`, any
`work/M*/milestone.md` (member tasks vs. Done-definition, planned vs.
actual capability drift), `work/design/design-review.md` if the project
went through `/design`, and `core/scripts/ledger`'s own
`aggregate`/`scan-untracked-ratio` numbers — the same instrument `/costs`
reports, framed the same non-leaderboard way. It writes
`docs/dashboard.html`.

As a side effect, any task missing its own `work/<task-id>/report.html`
gets one generated via `render-task` first, so every Gantt bar and event-
feed link the dashboard produces actually resolves.

Report the output path back to the human plainly: `wrote
docs/dashboard.html — open it in a browser`. **Never a gate** — not
invoked by `/task`, `/verify`, or `/ship`, and nothing downstream reads
the generated file back in. If `render-dashboard` itself could not run
(missing `jq`, not a git repo, unreadable input), say so directly; don't
retry silently or fall back to reconstructing the dashboard by hand.
