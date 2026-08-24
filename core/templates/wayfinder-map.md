<!--
One file per open or cleared effort: work/wayfinder/<map-id>/map.md.
Map IDs are `W1`, `W2`, ... — sequential, distinct in shape from every
other id spine uses (`M<n>` milestones, `D-<n>` decisions,
`<YYYYMMDD>-<slug>` tasks) for the same reason milestone ids are distinct
from task ids (core/templates/milestone.md's own header): nothing that
globs one shape should ever accidentally match another.

/wayfinder exists for the one gap neither /design nor /roadmap covers: an
effort whose destination is known but whose path is genuinely foggy —
too large and too undecided for one /design session's fixed six
categories, and not yet a known milestone list /roadmap could sequence.
This file is the map; core/scripts/wayfinder-frontier reads its Tickets
table mechanically to say what's resolvable next, exactly the same
"never hand-scan it, never trust a cached label" discipline
core/scripts/next-milestone-task already applies to milestone.md.

A ticket's own detail — the real question, and once resolved, the real
resolution and its evidence — lives in its own file,
work/wayfinder/<map-id>/tickets/<ticket-id>.md
(core/templates/wayfinder-ticket.md). This table is only ever that
ticket's mechanical summary: never write resolution prose here, and never
let this table's Status disagree with that file's own.

Three ticket types, each resolved by something spine already has —
/wayfinder never invents a fourth resolution mechanism:
  - grilling  — resolved by a facilitated conversation, in this session,
    the same "you propose, they decide" posture /design already uses.
  - prototype — resolved by a real /prototype session (the human types
    it themselves, same as any other real command) — the ticket stays
    open until they come back with its findings.
  - research  — resolved by delegating to the researcher agent
    (core/agents/researcher.md), unmodified, right in this session — a
    factual "how does X actually work today" question, not a judgment
    call, so it doesn't need the human present for the work itself, only
    for reviewing the outcome.

This is sequential multi-session coordination through committed files —
the same idiom /task already uses across a milestone's own member tasks,
just one level up, before the milestone list itself is known. It is not
team-of-agents orchestration: nothing here runs in parallel, there is no
"claim" mechanism, and at most one map is active at a time
(.spine/current-wayfinder) — see docs/tradeoffs.md's Deferred table for
what's still genuinely out of scope.
-->

# Wayfinder map `<map-id>`: <destination title>

## Destination

<!-- One paragraph: what this effort is trying to reach, and why it's big
     / foggy enough to need a map rather than going straight to /design or
     /roadmap. Fixed once, at map creation — if it needs to change, that's
     a real conversation with the human (append a dated note here, never
     silently rewrite the original), not a silent edit. -->

## Tickets

<!-- MACHINE: wayfinder-tickets

     One row per decision ticket this map has ever spawned. Never remove
     a row, even once resolved — core/scripts/wayfinder-frontier and any
     human skimming this file both need the full history, not just what's
     still open. core/scripts/wayfinder-frontier parses this table
     exactly; keep its column order and fence markers intact.

     Type: grilling | prototype | research
     Status: open | resolved  (never "blocked" — a ticket's blocked
       status is always derived from Blocked-by against the current
       resolved set, never hand-set; see core/scripts/wayfinder-frontier's
       own header for why a stored "blocked" label would drift)
     Blocked-by: comma-separated Ticket ids from this same table, or
       `none` — a map's tickets only ever block on each other, never on a
       ticket from a different map.
-->

| Ticket | Type | Status | Blocked-by | Question |
|---|---|---|---|---|
| T1 | grilling | open | none | <one line — the ticket file has the rest> |

<!-- next-ticket-id: 2 -->

<!-- Monotonic — never derived from "highest id currently present."
     Allocating a ticket id increments it; nothing ever frees or reuses
     one, same reasoning core/templates/milestone.md's next-gap-id
     already states for the identical drift risk. -->
