<!--
One file per project, at docs/product-spec.md. This is the primary
pre-charter handoff document — written by the engineer (or handed off from
whatever produced the product idea: your own notes, a Claude Design
session, a stakeholder brief) before or right after /bootstrap, any time
before /design first runs.

Consumers, real and exact:
  - /bootstrap tells the engineer to save a product spec/design handoff/
    build plan here by exactly this path (core/skills/bootstrap/SKILL.md).
  - /design auto-detects this file every run (not just the first) and
    offers to read it as --handoff grounding — it's read alongside
    docs/charter.md so the six foundational categories (state-management,
    persistence, module-boundaries, error-handling, auth-model,
    repo-topology) get proposed from what this document actually says,
    not guessed at cold. Section headers below are named to match that
    vocabulary exactly so a citation back to this file is precise, not
    "see the spec somewhere."
  - /wayfinder reads it (alongside docs/charter.md and docs/map.md/
    docs/design-summary.md, if they exist) as grounding when proposing a
    new map's initial ticket set — the "Known unknowns" section below is
    written in wayfinder's own three ticket types for exactly this reason.

Nothing here is authoritative on its own — this is grounding material a
human still confirms or revises through /design's and /wayfinder's own
propose/confirm loops, the same way a charter line is cited, not copied.
docs/charter.md itself stays the terse, ≤100-line, human-owned record of
what's actually confirmed; this file can be longer and rougher, and can
disagree with itself in places (a real spec often does before someone
sits down and decides) — that's fine, /design's job is to notice and ask.

**Do not hand-extract docs/vision.md from this file yourself.** A
milestone list assembled cold from a spec, before /design has run, skips
the six-category grounding check and the wayfinder ticket triage — exactly
the failure mode /wayfinder exists to prevent (core/skills/bootstrap/
SKILL.md says this explicitly). The "Build order" section below is raw
material /design or /wayfinder will turn into real docs/vision.md entries
— write it as your best current guess, not as a document you expect
anyone to copy verbatim.

Thin or empty sections are fine — don't manufacture content to look
thorough. A two-person weekend project's spec might be 60 lines; a real
product's might be 400. Length should track how much you actually know,
not a target.
-->

# Product spec: `<project-name>`

<!-- Optional. Useful once this file gets re-read as a --handoff for a
     later reconciliation pass — helps a human (and the design-registry-
     diff check, if this project has one) tell "this changed since last
     read" from "still the same document." -->
Last updated: `<yyyy-mm-dd>`

## What this is

<!-- One paragraph, plain language: what it does, who it's for, why it
     needs to exist. (→ charter's "What this product is") -->

## Primary users & core use cases

<!-- Not one of charter's own four buckets — kept here because "who
     actually uses this and for what" is exactly the kind of concrete
     detail that makes a proposed answer to state-management/persistence/
     auth-model land on the first try instead of the third. 3-6 real
     scenarios, one line each: "<user type> does <action> to get <outcome>."
     Skip entirely for a project too early to know yet. -->

## Non-negotiables

<!-- Properties that must hold regardless of what any single task or
     milestone asks for. Falsifiable — something a reviewer or an
     adversary agent could point at and say "this change violates this
     line." (→ charter's "Non-negotiables") -->

-

## Hard constraints

<!-- Real limits: regulatory, contractual, platform, org-boundary — not
     preferences. (→ charter's "Hard constraints") -->

-

## Explicitly out of scope

<!-- What this product is not trying to be, at least not yet. Keeps
     "why don't we just—" conversations short later. (→ charter's
     "Explicitly out of scope") -->

-

## Sequencing constraints

<!-- Build-order rules that are neither an invariant nor a limit, just a
     process fact — "X must exist before Y can be built or shipped."
     (→ charter's "Sequencing constraints") Empty is a legitimate answer. -->

-

## Glossary

<!-- Domain terms a research/plan/design doc can reference without
     re-explaining. (→ charter's "Glossary") -->

-

## Architecture leanings

<!-- Your best current answer for each of the six categories /design
     will actually walk — even a rough or uncertain one is useful raw
     material; /design proposes from this and the human confirms or
     revises every single one, it never adopts a leaning silently. Don't
     leave a category blank just because you're unsure — write the
     uncertainty down ("probably X, but Y might force otherwise because
     Z") rather than omitting it; that's more useful to /design than
     silence. Use exactly these six names — /design does not recognize a
     seventh. -->

**State management:**

**Persistence:**

**Module boundaries:**

**Error handling:**

**Auth model:**

**Repo topology:** <!-- single repo, or multiple repos coordinated via
     /workspace (core/skills/workspace/SKILL.md) — say which, and why,
     if you already know. -->

## UI/visual handoff

<!-- Only if this project has a rendered UI (browser-rendered, or native
     mobile/desktop via simulator/emulator) and a separate UI handoff
     bundle exists or is planned (tokens, component library, per-screen
     specs + screenshots — see core/templates/ui-handoff.md, meant to
     live at docs/ui/handoff.md). Point at it here rather than
     duplicating its content; the six categories above are architecture,
     this is the visual system, and they're deliberately kept as two
     different documents so each stays legible on its own. If this
     product's UI is already fully designed before implementation starts
     (e.g. a full Claude Design pass), say so and add "design system/
     component library built before any screen" to "Sequencing
     constraints" above — that ordering matters enough to state as a real
     constraint, not just implied by having a handoff bundle. "None — no
     rendered UI" or "None yet — planned" are both complete answers. -->

## Build order

<!-- Rough phase list, in the order you currently think they should ship,
     with a one-line "why this order" each. This is raw material for
     /design (milestone 0) and /wayfinder/roadmap after it — NOT a
     document to hand-copy into docs/vision.md yourself; see this file's
     own header above for why. -->

1.
2.

## Known unknowns

<!-- The highest-leverage section for /wayfinder specifically: split by
     how each would actually get resolved, using wayfinder's own three
     ticket types exactly (grilling | prototype | research) so a map
     seeded from this file starts with correctly-typed tickets instead of
     everything defaulting to "grilling." Skip a subsection entirely if
     you have nothing of that type. -->

**Needs a conversation** (an architecture/tradeoff call a human can just
decide, now or in a future session):

-

**Needs a prototype** (a real visual or behavioral question — "what does
this actually feel like" — that discussion alone can't settle):

-

**Needs research** (a factual question about how something — existing
code, a third-party API, a real constraint — actually behaves, not a
judgment call):

-
