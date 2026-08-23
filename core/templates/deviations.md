<!--
One `## Deviation N` record per entry, in the exact field shape below —
`/ship` greps this file for the literal line `- Status: open` and refuses
to ship while any exist — the merge gate: open deviations block. The
`- Status: <value>` line must contain exactly the
word `open` or `resolved` after the colon-space — no backticks, no other
punctuation, nothing else on the line — or the mechanical check can't see
it. Same for `- Tier:`, machine-read by nothing today but kept clean for
the same reason.

The circuit breaker: the third deviation of any kind,
tier notwithstanding, invalidates the current plan. Task state returns to
`research` — three wrong guesses means the research was wrong once, not that
each guess gets patched forward individually.
-->

# Deviations: <task-id>

## Deviation 1

- Tier: decide-alone | record-and-proceed | halt  <!-- pick exactly one word, delete the others -->
- Status: open | resolved  <!-- pick exactly one word, delete the other -->
- Plan assumed: <what the plan said would be true>
- Actually true: <what's actually true, with evidence — file:line or a
  command and its output, not a hunch>
- Options considered: <the real alternatives, briefly>
- Recommendation: <what this record proposes, and why>
- Resolution: <how it was resolved, dated. If this cites a
  `docs/charter.md` line, this line must record which of the two happened —
  the charter was edited (link the change) or explicitly reaffirmed as-is —
  never left implicit.>
