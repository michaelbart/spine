<!--
One file per big feature or big change, at docs/features/<feature-slug>-
handoff.md — the mid-project counterpart to docs/product-spec.md
(core/templates/product-spec.md). Use that one when founding a project;
use this one when a project is already installed, already has a confirmed
docs/charter.md and real docs/decisions/, and you're about to hand off
something too big to just type as a /task description.

This file does not get auto-detected by anything the way docs/
product-spec.md does — there's no single fixed downstream command,
because which command is right depends on what the feature actually is.
That's what section 2 below (Routing) is for: decide it once, up front,
instead of re-deriving it in whichever skill session picks this up.

Nothing here is authoritative on its own — same posture product-spec.md
takes. Every section below is grounding a human still confirms through
whichever command Routing points at: /design's propose/confirm loop,
/wayfinder's ticket confirmation, or /roadmap's sequence confirmation.

Don't hand-extract docs/vision.md yourself even for a feature-sized
effort if its shape is still genuinely foggy — same reasoning
product-spec.md's own header states, unchanged by scale.
-->

# Feature handoff: `<feature name>`

Last updated: `<yyyy-mm-dd>`

## 1. Feature summary

<!-- One paragraph: what this is, why now. Assume the reader already
     knows the product (docs/charter.md, docs/design-summary.md) — don't
     re-explain it, just this feature's own shape and motivation. -->

## 2. Routing

<!-- The single most useful section in this file: decide, right now,
     which real command this handoff feeds — so whoever (or whichever
     session) picks this up next doesn't have to re-derive it. Delete the
     three options you're not choosing; keep the one that applies, filled
     in. If you're genuinely unsure, say so and default to the foggiest
     option below (/wayfinder) rather than guessing small — /wayfinder's
     own §1 sanity check will point back to /roadmap or /design if this
     turns out to be simpler than it looked, and that redirect is cheap;
     committing to a milestone list that turns out to hide a real
     architecture question is not. -->

**This feature:**

- [ ] **Is architecture-shaped** — it revises or extends a foundational
  decision (state-management, persistence, module-boundaries,
  error-handling, auth-model, repo-topology), not just a chunk of
  feature work. → Run `/design --handoff <path to this file>`
  (`core/skills/design/SKILL.md`'s handoff re-entry mode) before anything
  else — it reads §4 below to decide, per finding, whether to supersede an
  existing decision or write a new one. Sections 5-7 still matter as
  grounding for whatever milestone work follows design review.

- [ ] **Has a known milestone shape already** — you can already write the
  ordered phase list, no open architecture question, no real unknowns
  beyond normal implementation detail. → Skip design entirely. Feed §6
  (Build order) straight into `docs/vision.md`'s `## Planned milestones`
  and run `/roadmap` to sequence it.

- [ ] **Is genuinely foggy** — you know the destination, not the path, and
  it's bigger than one `/design`-style sitting. → Hand `/wayfinder` §1's
  destination prompt the text of §1 above; let §7 (Known unknowns) seed
  the map's initial ticket set instead of proposing it cold.

- [ ] **Is actually task-sized** — on reflection this doesn't need a
  handoff document at all. → Stop here, run `/task <description>`
  (or `/intake <ticket>` if it's tracked) directly.

## 3. Fit within the existing charter

<!-- Quote or closely cite the specific docs/charter.md line(s) this
     feature must respect. If it would actually change a charter line
     (a non-negotiable, a hard constraint, an out-of-scope item) — say so
     explicitly and propose the edit; a charter change is real, human-
     owned, and dated, never something a downstream command infers and
     applies silently (docs/charter.md's own header). -->

## 4. Relationship to existing decisions

<!-- Cite every docs/decisions/D-<n>-*.md this feature touches, extends,
     or plausibly conflicts with, by id. For each: does it fit within the
     decision as adopted, or does it need to supersede it? This is exactly
     what /design --handoff re-entry mode classifies against (its own
     "Classify the handoff's content, don't re-interview" step) if
     Routing above pointed there — write it precisely enough that step
     doesn't have to re-derive it from prose. "None — no architectural
     overlap" is a complete, correct answer for a pure feature addition. -->

## 5. Scope & non-goals for this feature

<!-- The feature-scoped version of charter's "Explicitly out of scope" —
     what this effort is deliberately not doing, even if adjacent and
     tempting. -->

-

## 6. Build order

<!-- Only if Routing above chose the known-milestone-shape path. Ordered
     phase list with a one-line "why this order" each — this is what gets
     copied into docs/vision.md's `## Planned milestones`, not a separate
     document to keep in sync with it. Delete this section if Routing
     chose /design or /wayfinder instead. -->

1.
2.

## 7. Known unknowns

<!-- Only if Routing above chose the foggy/wayfinder path. Same three
     types wayfinder itself uses (core/skills/wayfinder/SKILL.md §1) —
     type each one now so the map's first pass starts correctly typed:

     grilling  — resolvable by a facilitated conversation.
     prototype — resolvable only by building something concrete and
                 seeing/using it — a real visual/behavioral unknown.
     research  — resolvable by investigating how existing code actually
                 behaves today — a factual question, not a judgment call.

     Delete this section if Routing chose /design or /roadmap instead. -->

**Needs a conversation:**

-

**Needs a prototype:**

-

**Needs research:**

-

## Source materials

<!-- Real paths this handoff was distilled from — a ticket, a prior
     discussion doc, a Claude Design export under docs/ui/, meeting
     notes. "None" is fine for something written fresh. -->

Source documents: none | `<path>`[, `<path>`...]
