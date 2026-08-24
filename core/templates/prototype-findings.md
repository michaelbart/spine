<!--
One file per prototype session: work/prototypes/<prototype-id>/findings.md.
/prototype is spine's declared escape hatch for the "I don't know what I
want yet" spike docs/tradeoffs.md's "Where this is the wrong tool" section
used to warn away from spine entirely — narrow and bounded on purpose: no
class, no plan, no floor, no adversary review, no `Spine-Task:` trailer.
Its only job is producing a concrete, disposable artifact that answers one
question discussion alone couldn't settle, and this file, which records
what was learned so the answer outlives the artifact whether or not the
artifact itself is kept.

This file is real citable evidence once written — a /design decision's
Context section, a /wayfinder ticket's Resolution, or a /task's plan can
all point at it — but it is never itself one of the three entry paths
docs/decisions/decision.md's own header names for writing a decision
record. A prototype settles a question; the human still decides, through
one of those three real paths, whether and how that answer becomes a
recorded decision.
-->

# Prototype `<prototype-id>`: <one-line question>

- Question: <the question this prototype exists to answer, one line>
- Started: <yyyy-mm-dd>
- Disposition: kept | discarded

## What was built

<!-- One or two sentences: the concrete thing built to answer the
     question — not a design rationale, just what exists (or existed) at
     work/prototypes/<prototype-id>/. -->

## What was learned

<!-- The actual answer, stated as a plain conclusion — "modal confirmation
     reads as an interruption in this flow, inline confirmation doesn't"
     — not a narrative of the building process. This is the part that
     outlives the artifact. -->

## Feeds into

<!-- Where this answer goes next, if anywhere: a specific /design
     foundational category, a specific /wayfinder ticket id, a specific
     task's plan.md, or "nothing yet — recorded for later reference." Real
     and specific, never invented — not every prototype resolves
     something formal, and that's a legitimate outcome. -->
