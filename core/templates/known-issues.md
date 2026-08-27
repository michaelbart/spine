<!--
One file per project: docs/known-issues.md. A low-ceremony ledger for
observations discovered outside any active task or ticket — manual
testing, a hunch, something noticed in passing — with no active `/verify`
run and no ticket-tracker key to attach it to (`/intake` requires a real
ticket, which a local-only project won't have). Populated by
`/note-issue <one-line>` (core/skills/note-issue/SKILL.md), which wraps
`core/scripts/issue-ledger add`.

This is a structurally separate file from any `work/<milestone-id>/
milestone.md`'s own "## Known gaps for future member tasks" section. That
section is provenance-locked to adversary-verdict-sourced entries —
populated only by `/ship`'s flagged-finding triage, "never hand-invented
speculatively" (core/templates/milestone.md) — and this file must never
be confused with it, merged into it, or treated as an alternate way to
populate it. An entry here has no adversary verdict behind it; it's a
human (or an agent, on the human's behalf) noting something worth
tracking later, nothing more, and it carries no claim that it traces to a
plan's acceptance check the way a Known-gap does.

Shape per entry — same fenced, machine-parseable discipline as
Known-gaps, deliberately:

- id: issue-<n>
  severity: low | med | high
  status: open | resolved
  noted_at: <iso8601>
  evidence: <file:line | url | free text, or "none" if nothing more specific>
  <one-line description, verbatim as given to /note-issue>

`next-issue-id` (below) is a monotonic counter, never derived from
"highest id currently present" — a resolved entry must never free its id
for reuse, same discipline as Known-gaps' `next-gap-id` and for the same
reason: a stale citation to a resolved `issue-3` and a new, unrelated
`issue-3` would be genuinely ambiguous later.

`/roadmap`'s §1b absorbs every *open* entry here into a milestone, or an
explicit human decline, every time `/roadmap` runs — mirroring §1a's
Known-gaps discipline exactly: never silently dropped. `/spine` surfaces
the open-issue count so entries can't rot invisibly between `/roadmap`
runs. Manage this file only through `core/scripts/issue-ledger` (`add` /
`list` / `resolve`) or `/note-issue` — hand-editing an entry's `id` /
`status` / `noted_at` desyncs it from what those tools expect to find.
-->

# Known issues

<!-- MACHINE: known-issues -->

<!-- next-issue-id: 1 -->
