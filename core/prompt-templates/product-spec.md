<!--
Use the main prompt below in a fresh Claude chat session (or any capable
model), before or right after /bootstrap, any time before /design has
run. Have the conversation, then save the assistant's final output as
docs/product-spec.md verbatim — /design auto-detects that exact path.

If you already have raw material (notes, a voice memo transcript, a prior
spec, a pitch deck's text) paste it in after the prompt so the session
drafts from real material instead of interviewing you cold.

**If your idea currently exists only inside a Claude Design session** —
no separate notes, the design itself is the only record of the idea —
run Stage 0 below there first, save its output, and paste that in as your
"raw material" for the main prompt. Don't skip straight from Claude
Design to a finished product-spec.md: a visual design is strong evidence
for what the app does and who it's for, but it has no way to tell you
your non-negotiables, hard constraints, or architecture leanings — those
need a real conversation, not an inference from screens. Stage 0 exists
specifically to hand the chat session honest raw material instead of
asking it to reverse-engineer business context from a UI it has no
grounding to interpret.
-->

# Stage 0 — only if your idea currently exists solely in Claude Design

Run this in that same Claude Design session before doing anything else,
then save its output — you'll paste it into the main prompt below.

```
Write a plain-language design summary of the app you just built here —
prose a human will read once, not a technical export (that comes later,
separately, for the actual UI handoff). Cover:

1. What the app does and who it's for, in your own words, based only on
   what you actually designed — don't invent context beyond the screens.
2. Every screen: its purpose, and the user action(s) that lead into and
   out of it.
3. Every business rule the design implies, even implicitly — e.g.
   "checkout is gated behind login," "a draft can't be published until
   field X is filled," "an admin role sees a different dashboard than a
   regular user." Cite which screen or interaction each rule comes from.
4. Every domain term the UI's own copy uses more than once (e.g.
   "listing," "workspace," "run") — a one-line definition each, based on
   how it's actually used in the design.
5. Explicitly flag anything you genuinely can't tell from the design
   alone — data retention rules, who else can see what, what happens on
   error, whether this needs to work offline, compliance requirements,
   how it's hosted, how many people will use it. Don't guess at these —
   list them as open questions instead, so I know to bring them up
   myself in the next conversation.

Give me this as one plain-text document I can paste elsewhere, not a
file export.
```

# Prompt: draft a spine product spec

Paste Stage 0's output (if you ran it) right after this prompt, labeled
"Design summary from Claude Design:" — tell the chat session explicitly
that it's evidence for section 2 (primary users & use cases) and section
7 (glossary) below, not an authoritative source for sections 3-4 and 8
(non-negotiables, hard constraints, architecture leanings) — those still
need to come from you, in this conversation, even if the design summary
seems to imply an answer.

```
I'm about to start a new software project that will be built using
"spine," an AI-development workflow with a fixed planning vocabulary. I
need you to interview me and then draft docs/product-spec.md in the exact
structure below — this becomes the grounding document a later planning
stage reads to propose architecture decisions and a milestone sequence,
so precision in each section matters more than polish.

Interview me conversationally, one topic at a time, don't dump all these
questions at once. Push back if an answer is vague ("scalable" isn't a
non-negotiable — what specifically must hold?) or contradicts something I
said earlier. When we're done, or when I say "draft it," produce the full
markdown document.

If I've pasted a "Design summary from Claude Design" below: treat it as
real evidence for section 2 (primary users & use cases) and section 7
(glossary) only — draft those from it rather than asking me to repeat
what it already says. For every other section, especially 3, 4, and 8
(non-negotiables, hard constraints, architecture leanings), still
interview me directly, even where the design summary seems to imply an
answer — a screen showing a login flow tells you there's an auth
experience, not what the auth model should be, and a visual design can't
tell you a regulatory constraint at all. If the summary's own "explicitly
flag" section lists open questions, raise every one of those with me by
name before we finish.

Required sections, in this order, with this exact meaning:

1. **What this is** — one paragraph: what it does, who it's for, why it
   needs to exist.
2. **Primary users & core use cases** — 3-6 concrete "<user type> does
   <action> to get <outcome>" scenarios. Skip if genuinely too early to
   know.
3. **Non-negotiables** — properties that must hold no matter what any
   future task or milestone asks for. Each one must be falsifiable —
   something a reviewer could point at a change and say "this violates
   this line."
4. **Hard constraints** — real limits only: regulatory, contractual,
   platform, org-boundary. Not preferences.
5. **Explicitly out of scope** — what this is deliberately not trying to
   be, at least not yet.
6. **Sequencing constraints** — real "X must exist before Y" build-order
   rules, if any. Empty is a legitimate, correct answer.
7. **Glossary** — domain terms worth defining once.
8. **Architecture leanings** — my best current guess for each of these
   six categories, exactly these names, one subsection each:
   state management, persistence, module boundaries, error handling,
   auth model, repo topology (single repo, or multiple coordinated
   repos). A rough or uncertain guess is fine and expected — write the
   uncertainty down ("probably X, but Y might force otherwise because Z")
   rather than picking one to sound decisive. Don't skip a category for
   being unsure.
9. **UI/visual handoff** — ask whether this project has a rendered UI at
   all (browser-rendered, or native mobile/desktop via simulator/emulator),
   and if so whether a separate design handoff (tokens, component
   library, per-screen specs, screenshots) exists or is planned. If it's
   fully designed up front (e.g. a Claude Design pass before any code),
   say so explicitly and add "design system built before any screen" to
   section 6 above.
10. **Build order** — a rough phase list, in the order I think they
    should ship, one line of "why this order" each. Make clear in the
    document that this is raw material for a later planning stage, not a
    milestone list to copy verbatim anywhere.
11. **Known unknowns** — split into exactly three buckets, and ask me
    enough to actually sort real items into each rather than defaulting
    everything into one bucket:
    - Needs a conversation (a judgment call a human could just decide)
    - Needs a prototype (a real visual/behavioral question discussion
      alone can't settle — "what does this actually feel like")
    - Needs research (a factual question about how something — existing
      code, a third-party API, a real constraint — actually behaves)

Thin or empty sections are fine and expected in places — don't invent
content to look thorough. Match section length to how much I actually
know, not a target length.
```
