<!--
One file per project: docs/design-summary.md. Written once, at the end of
/design (§7.5, after design review resolves and decisions adopt, before
the commit in §8) — never hand-maintained afterward. If a later decision
supersedes one summarized here, the next /design re-run (or a deliberate
/ratchet-style pass) rewrites this file; nothing else keeps it current, so
treat a stale-looking summary as a signal to regenerate, not a bug to
patch by hand.

This exists because docs/decisions/D-*.md records are optimized for
machine consumption (decision-hash, verdict-filter citation resolution,
check-stale grounding) — precise, but not something a human reads
start-to-end to get the shape of what was decided. docs/map.md doesn't
fill this gap either: it's /remap-generated post-code and empty at design
time. This file is the missing "what did we actually land on, and why"
a human skims once before their first milestone task, not a replacement
for the decision records themselves — always point back to the real
D-<n> record for provenance/alternatives-considered, don't duplicate it.

Audience is a human engineer about to start work, not a future adversary
agent or a mechanical check — write in plain sentences, no jargon a
newcomer to this specific project wouldn't have, no decision-record
scaffolding (no "- Status:", no "## Alternatives rejected" headers). Short
enough to actually get read: a paragraph per foundational category, not a
page. If a category's paragraph is fighting to stay under 5 sentences,
that's a sign the decision itself may be doing too much — flag it, don't
just write a longer paragraph.
-->

# `<project-name>`: what we decided and why

<!-- 2-3 sentences, plain language: what this product is, restated from
     the charter's own "What this product is" section but in your own
     words, not copy-pasted. A reader who has never opened charter.md
     should understand the product from this alone. -->

## The shape of it

<!-- One short paragraph per adopted decision, grouped by the six
     foundational categories in this fixed order: state-management,
     persistence, module-boundaries, error-handling, auth-model,
     repo-topology. Skip a category entirely (no heading, no paragraph)
     if it has no adopted decision and only a DEFERRED.md entry instead —
     list those under "Open questions" below, don't pad this section with
     "not yet decided" paragraphs.

     Each paragraph: what was decided, in one or two sentences of plain
     prose (not the decision record's own "Decision" text verbatim — a
     human-readable restatement) — then one sentence on why it matters
     practically for someone about to write code here ("this means your
     component X should never do Y"). End with the decision id in
     parens so a reader who wants the full record (context, rejected
     alternatives, consequences) knows exactly where to look — `(see
     D-<n>)` — never repeat that record's content here. -->

### State management

### Persistence

### Module boundaries

### Error handling

### Auth model

### Repo topology

## Open questions

<!-- One line per docs/decisions/DEFERRED.md entry, in plain language:
     what's still open and what event will force deciding it. "None" only
     if DEFERRED.md is genuinely empty. -->

## Where to look next

<!-- Fixed, short pointer list — this section's content doesn't vary by
     project beyond the paths:
     - Full decision records with rejected alternatives: `docs/decisions/D-*.md`
     - What was reviewed and how findings were resolved: `work/design/design-review.md`
     - The walking-skeleton milestone: `work/M0/milestone.md`
     - The non-negotiables everything above traces back to: `docs/charter.md` -->
