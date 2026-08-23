---
name: security-checklist
description: Background checklist preloaded into the security adversary agent. Not a user command.
user-invocable: false
---

Checklist for attacking a diff, scoped to the three dominant defect
categories — authorization, data integrity, injection.
Stack-blind: no tool or language is named below; read the actual mechanism
in the diff, don't pattern-match on syntax.

Every claim you file must carry evidence per `core/ADAPTER-CONTRACT.md §5`
(a `file:line` or a command plus its captured output) or `verdict-filter`
drops it before `/verify` ever sees it — an unevidenced claim is wasted
effort, not a lower-confidence finding.

## Authorization

- For every new or changed entry point (route, handler, query, mutation):
  what proves the caller is allowed to do this, specifically — not "this
  file is behind login," but this action, on this record, for this caller.
- Ownership/tenant checks: does the check compare the caller's own
  identity/org/tenant against the record's, or does it just check the
  caller is *some* authenticated identity? The second is the common defect.
- Checks performed client-side, or in a layer the caller map (from `floor`'s
  `callers` artifact) shows is reachable another way that skips them.
- Elevation paths: anything that changes a caller's own role, permissions,
  or another user's access.
- Bulk/list endpoints: does filtering happen in the query, or after
  fetching data the caller shouldn't have received at all?

## Data integrity

- Concurrent-write races on anything the plan's acceptance checks assume is
  atomic — construct the interleaving, don't just assert one exists.
- Partial-failure states: what does the system look like if this change's
  operation fails halfway, and does anything downstream assume it's all-
  or-nothing?
- Validation that exists on one path (e.g. a form) but not another (e.g. a
  direct API/import/batch path) reaching the same write.
- Backfills/migrations touched by this diff: does an invariant query exist
  and does it check *both* the old and new shape (see
  `core/rules/migrations.md`), not just the new one?

## Injection / trust boundary

- Anywhere external input (user input, another service's response, a file,
  a queue message) reaches a query, a shell command, a template, or a
  deserializer without going through this codebase's existing sanitization
  path — find the existing path first from the caller map, then check this
  diff actually uses it rather than reimplementing something weaker.
- New dependencies (per `dep-diff`'s artifact): what trust boundary does
  this one cross, and is its input from this change's diff or from
  further upstream, attacker-influenced data?

## Reporting

State what you attacked (`attacked`, non-empty even on a clean bill) before
you state what you found. A clean bill is a real result, not a non-result —
say specifically which of the above categories applied to this diff and
that you checked them, not "nothing to check here."
