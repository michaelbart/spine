<!--
One file per ticket: work/wayfinder/<map-id>/tickets/<ticket-id>.md. The
map.md table row (core/templates/wayfinder-map.md) is this ticket's
mechanical summary — status, type, blocked-by, the frontier script's whole
input. This file is where the real question and its resolution live,
citable the same way work/<task-id>/research.md or docs/decisions/D-<n>.md
already are — a decision record written from a resolved wayfinder ticket
(the human-directed entry path, docs/decisions/decision.md's own header)
should cite this file's path in its Context section, not just the map.
-->

# Ticket `<ticket-id>`: <one-line question>

- Type: grilling | prototype | research
- Status: open | resolved
- Blocked-by: <ticket-id>[, <ticket-id>...] | none
- Opened: <yyyy-mm-dd>
- Resolved: <yyyy-mm-dd> <!-- leave this line's value blank until Status: resolved -->

## Question

<!-- The real question, in full — what's actually undecided, and why it
     blocks the map's own destination (core/templates/wayfinder-map.md's
     `## Destination`). One or two paragraphs; the map.md table row is
     the one-liner, this is the real thing. -->

## Resolution

<!-- Blank until Status: resolved. Then, depending on Type:
       - grilling: the reasoning itself, written the way a design-review
         resolution is (core/skills/design/SKILL.md §6) — what was
         decided and why, not a transcript of the conversation.
       - prototype: a citation to work/prototypes/<id>/findings.md plus
         one sentence on what it settled — never restate the prototype's
         own findings here, cite them.
       - research: a citation to this ticket's own sibling file,
         work/wayfinder/<map-id>/tickets/<ticket-id>-research.md (the
         researcher agent's full reply, written verbatim), plus one
         sentence summarizing the conclusion this ticket needed from it.
     Never leave this section started but incomplete — a ticket is either
     still open, or has a real, complete resolution. Nothing in between. -->

## Spawned tickets

<!-- Ticket ids this resolution revealed as new open questions, or
     `none`. Each must also get its own row in map.md's table; mark a
     spawned ticket Blocked-by this one only if resolving this ticket was
     a genuine precondition for starting the new one, not just something
     noticed along the way. -->
