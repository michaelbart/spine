<!--
One file per project: docs/decisions/DEFERRED.md. One `## Deferred N` item
per deliberately-undecided foundational category, each naming the trigger
that forces deciding it. This is /design's most valuable section, not a
formality — a foundational category with neither an adopted decision nor
an entry here fails design-gate's stopping rule (check #3) by construction,
and an entry without a real, concrete `- Trigger:` line doesn't count as
one ("decide later" is not a trigger).

`- Category:` must be exactly one of the same six tokens
core/templates/decision.md's own `- Category:` field uses (plus `other`)
— design-gate checks #3 by confirming every one of the six is covered by
*either* an adopted decision carrying that Category *or* a DEFERRED item
carrying it, so the vocabulary has to match exactly, not just read as
equivalent to a human.

An empty file (nothing here, nothing decided either) fails the stopping
rule for every category at once — that is deliberate, not a bug in
design-gate; the stopping rule exists precisely to make "handed off
without deciding or deferring" impossible to do quietly.
-->

# Deferred decisions

## Deferred 1

- Category: state-management | persistence | module-boundaries | error-handling | auth-model | repo-topology | other
- Item: <what's left undecided, one line>
- Trigger: <the concrete event that forces deciding it — e.g. "first task needing offline support decides this">
