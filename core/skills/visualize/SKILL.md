---
name: visualize
description: Generate a project-wide HTML dashboard — task Gantt timeline, event feed, traced-trivial log, decision store, capability matrix, milestone progress, cost/drift instrument. Bounded window by default with client-side facet/filter/search. Read-only, never a gate.
argument-hint: [--project <path>] [--since <date> | --all]
---

You are running `/visualize`. `$ARGUMENTS` is an optional
`--project <path>` (default: current working directory), plus an optional
window flag: `--since <date>` or `--all`.

Confirm the resolved project root is spine-installed
(`.spine/capabilities.json` exists) — if not, say so and stop rather than
letting the script fail with a less legible message.

```
${CLAUDE_SKILL_DIR}/../../scripts/render-dashboard --project <project root> [--since <date> | --all]
```

**Windowing (dashboard at scale).** By default the dashboard renders a
*bounded window* so it stays usable after months of multi-engineer work:
every not-yet-done task, plus any task (or Class 0 trace, or non-task commit)
with activity in the last ~14 days. This is applied server-side to the
task-derived sections (Gantt strip, event feed, traced-trivial) — decisions,
the capability matrix, and milestones already render in full. A prominent
"Covers …; showing N of M" banner names the exact window so a windowed-empty
view is never mistaken for nothing-happened.

- `--since <date>` widens the window back to `<date>` (mirrors `/costs
  --since`; accepts `YYYY-MM-DD`, an ISO timestamp, or `N days|weeks|months
  ago`).
- `--all` renders the full history — every task and every event — reproducing
  the pre-windowing dashboard. Reach for it when someone needs older,
  already-shipped work back on the page.

Within whatever window is rendered, the HTML is self-contained and
CSP-safe (no external scripts/styles/fonts, no network): a filter bar facets
by owner / class / autonomy / status-or-phase / milestone / ticket, a
free-text box searches task ids, titles, ticket keys, and decision ids, a
landing summary recomputes counts as filters apply, completed tasks are
collapsed by default, and the event feed is grouped into per-day collapsible
sections. All client-side over the already-windowed set — no re-render.

This aggregates everything spine has already produced for the whole
project — every `work/<task-id>/` folder's class/state/verify result and
ledger phase timestamps (for the Gantt strip), the real git log (for the
event feed, flagging any `Spine-Bypass:` commit), `docs/decisions/D-*.md`
(with supersession chains), the live `.spine/capabilities.json`, any
`work/M*/milestone.md` (member tasks vs. Done-definition, planned vs.
actual capability drift), `work/design/design-review.md` if the project
went through `/design`, `.spine/trace.jsonl` (the traced-trivial / Class 0
log, shown as its own collapsible section when present), `.spine/profile.json`
(the active team strictness profile, named in the header so the drift numbers
have context — absent means the built-in `standard`), and
`core/scripts/ledger`'s own `aggregate`/`scan-untracked-ratio` numbers — the
same instrument `/costs` reports, framed the same non-leaderboard way. It
writes `docs/dashboard.html`.

As a side effect, any task missing its own `work/<task-id>/report.html`
gets one generated via `render-task` first, so every Gantt bar and event-
feed link the dashboard produces actually resolves.

Report the output path back to the human plainly: `wrote
docs/dashboard.html — open it in a browser`. **Never a gate** — not
invoked by `/task`, `/verify`, or `/ship`, and nothing downstream reads
the generated file back in. If `render-dashboard` itself could not run
(missing `jq`, not a git repo, unreadable input), say so directly; don't
retry silently or fall back to reconstructing the dashboard by hand.
