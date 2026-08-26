<!--
Use this prompt in a fresh Claude chat session (or any capable model),
before or right after /bootstrap, any time before /design has run. Have
the conversation, then save the assistant's final output as
docs/product-spec.md verbatim — /design auto-detects that exact path.

If you already have raw material (notes, a voice memo transcript, a prior
spec, a pitch deck's text) paste it in after the prompt so the session
drafts from real material instead of interviewing you cold.
-->

# Prompt: draft a spine product spec

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
9. **UI/visual handoff** — ask whether this project has a browser UI at
   all, and if so whether a separate design handoff (tokens, component
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
