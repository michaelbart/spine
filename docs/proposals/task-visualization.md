# Proposal: spine task/milestone visualization

Status: v1 IMPLEMENTED — `core/scripts/render-task` + `/task-report`
(2026-08-15). The four open questions below were resolved directly with
the user: per-task view only (no milestone swimlane — v2, still open),
output at `work/<task-id>/report.html` (git-tracked), on-demand only
(never wired into `/ship`), built in spine core rather than
`g1-tee-waitlist`.
Origin: raised mid-session while shipping `g1-tee-waitlist`'s
`20260815-ui-render-adapter` task (see `~/spine/docs/tradeoffs.md`'s
g1-tee-waitlist testbed findings for the session this came out of).

## The problem

Right now the only way to understand what happened on a task is to read
`work/<task-id>/{research,plan,deviations,verify,briefing}.md` in order,
by hand. That's fine for one task read once, but it doesn't scale to:

- Getting the gist of a task fast (what changed, why, what surprised us,
  did anything get caught).
- Seeing a milestone's shape across its member tasks (what's done, what's
  blocked, what's still `TBD`).
- Seeing what the adversaries (falsifier/security) actually did — not
  just the final verdict, but the `attacked` list and which findings
  survived `verdict-filter` vs. got dropped.

The user's own framing, verbatim from the session: *"it would be really
neat if the spine project included some way to visualize everything that
has been done for the project being worked on... a UI that can be opened
up (html or something)... it will help give context, and understanding
to what it's doing and in what order and why it's beneficial."* Follow-up:
*"it would be nice to see at a glance/overview what the reviewers/
security&falsifier agents/etc. do."*

## What already exists (nothing new to instrument)

Every input this needs is already produced by `/task`/`/verify`/`/ship`,
structured and machine-readable:

- `work/<task-id>/ledger.json` — phase timestamps, token costs per phase,
  `class_declared`, `deviation_count`, `conformance_score`, etc.
- `work/<task-id>/plan.md` — `<!-- MACHINE: header -->` (class, owner,
  grounding), `## Predicted touch` (machine-fenced), `## Grounds on
  decisions`.
- `work/<task-id>/deviations.md` — one `## Deviation N` per record, fixed
  fields (`Tier`, `Status`, `Plan assumed`, `Actually true`, `Resolution`).
- `work/<task-id>/verify.md` — floor results table, `## Adversary
  verdicts` (per-adversary `attacked` list + kept verdicts + kept/dropped
  counts), capability/tooling gaps.
- `work/<task-id>/artifacts/<agent>-verdict.json` — the actual structured
  verdict objects `verdict-filter` kept (schema: `ADAPTER-CONTRACT.md §5`).
- `work/<task-id>/briefing.md` — the one-page human summary, already
  distilled.
- `work/<id>/milestone.md` — `## Member tasks` (with real task IDs once
  assigned), `## Inter-task contracts`, `## Capability targets`,
  `## Done-definition`.
- `docs/decisions/D-<n>-*.md` — decision records, with `## Implementing
  paths` naming every task that touched them.

This is the entire point of the proposal: **no new instrumentation, only
rendering** of data that's already structured and already correct.

## Design

A new skill/script — working name `/task-report` or `/visualize` — that
reads one task's (or one milestone's) `work/` folder and generates a
single self-contained static HTML file. Not a live server, not a
dashboard that polls — a generated artifact, regenerable on demand, same
posture as `docs/map.md`/`/remap` (deterministic output from static
input, never hand-maintained).

### Per-task view

- **Timeline strip** across the top: classify → research → plan →
  implement → verify → ship, each phase labeled with its ledger timestamp
  and token cost. Click/hover a phase to jump to its section.
- **The gist** — lifted verbatim from `plan.md`'s `## The gist` and
  `briefing.md`'s `**What & why**` — this is the "why does this task
  exist" answer, shown first, not buried.
- **Deviations** — one card per `deviations.md` record: tier (color-coded:
  decide-alone=neutral, record-and-proceed=amber, halt=red), status,
  "plan assumed" vs. "actually true," resolution. This is literally the
  "what surprised us" data, already structured.
- **Adversary activity** (the user's own explicit follow-up ask) — one
  panel per adversary that ran:
  - The `attacked` list, verbatim — what was actually checked, not
    inferred.
  - Every verdict, grouped kept-vs-dropped, severity-colored
    (high=red, medium=amber, low=gray), each with its evidence
    (`file_line` renders as a clickable path:line-style label; `command`
    renders as a collapsible command+output block).
  - A one-line summary badge: "6 kept, 0 dropped, max severity: medium."
- **Floor results** — the capability table from `verify.md`, pass/fail/
  degraded, color-coded.
- **Decisions grounded on** — links out to the actual `docs/decisions/
  D-<n>-*.md` files this task cites or distills.

### Per-milestone view

- **Member task list** as a horizontal swimlane/kanban-style strip: each
  member task is a card showing its own id, class, current `state`, and
  a mini status chip (done/in-progress/TBD). Clicking a card opens (or
  links to) that task's own per-task view.
- **Inter-task contracts** rendered as a simple dependency graph or
  annotated list: "task 1 leaves behind X, which task 2's plan
  references" — this is already prose in `milestone.md`, just needs
  visual grouping, not new data.
- **Capability targets vs. done-definition** — a simple table:
  capability name, planned status, current real status (read live from
  `.spine/capabilities.json`, not frozen at milestone-authoring time),
  so drift between "planned" and "actual" is visible without cross-
  referencing two files by hand.

## Implementation approach

- A new script, `core/scripts/render-task`, that takes `--task <id>` or
  `--milestone <id>` plus `--project <path>`, reads the relevant `work/`
  files (plain text/markdown parsing — the machine-fenced sections
  already have stable, documented grammars per `core/templates/plan.md`'s
  own header comments), and writes a single HTML file to
  `work/<id>/report.html` (or a milestone-level path).
- No new framework dependency — inline CSS/JS in the generated HTML,
  same self-containment discipline the `Artifact` tool already enforces
  elsewhere in this ecosystem. A generated file should be viewable by
  opening it in a browser with zero server, zero build step.
- A thin skill wrapper (`/task-report <task-id>` or similar) that calls
  the script and tells the human where the file landed — mirrors how
  `/remap` regenerates `docs/map.md`.
- **Never a gate.** This is purely observational tooling, like `/costs`
  — it never blocks `/verify`/`/ship`, and nothing else ever reads its
  output back in (same "pure human output" property `briefing.md`/
  `pr-description.md` already have, confirmed via
  `work/.build/readability-phase-A-handoff.md`'s consumer audit — worth
  re-confirming for this new artifact rather than assuming it transfers).

## Open questions (need the user's decision before scoping this as a real task)

1. **Scope of v1**: per-task view only, or per-task + per-milestone in
   the same pass? (Per-task is the smaller, faster win; per-milestone
   needs the swimlane logic and is where the "in what order and why"
   framing from the original ask lives most directly.)
2. **Where does the generated file live?** `work/<task-id>/report.html`
   (co-located, git-tracked like everything else in `work/`) vs. a
   `.gitignore`d scratch location (regenerate on demand, never committed)?
   Precedent split: `briefing.md`/`verify.md` are committed;
   `.spine/.smoke-runtime-seconds` is a machine-local cache. This is
   closer to the latter in spirit (fully derivable from already-committed
   data) but closer to the former in that a teammate reading the repo
   might want it without regenerating.
3. **Auto-generate at `/ship` time, or on-demand only?** Auto-generating
   at ship time makes it always current for free, but adds a step to
   every ship. On-demand keeps `/ship` unchanged but means it can go
   stale/never get made.
4. **Cross-task aggregate view** (e.g. "show me every task in the last
   week") — explicitly out of scope for v1 per this proposal, but worth
   naming as a natural v2 extension once `/costs`' own aggregate patterns
   exist to borrow from.

## Non-goals (stated explicitly so this doesn't scope-creep)

- Not a live dashboard — no polling, no server, no auto-refresh.
- Not a replacement for `briefing.md`'s prose — the visualization
  surfaces structure faster, it doesn't replace the "why this mattered"
  narrative a human wrote.
- Not a new instrumentation layer — if a future need requires data that
  doesn't already exist in `work/<task-id>/*`, that's a different
  proposal (extend the ledger/verify.md schema first), not something
  this visualization tool should special-case around.
