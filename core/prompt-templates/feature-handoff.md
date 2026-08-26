<!--
Use this prompt in a fresh Claude chat session, mid-project — after
docs/charter.md and at least some docs/decisions/D-*.md already exist.
Paste in whatever raw material prompted the feature (a ticket, a
discussion thread, notes) after the prompt. Save the output as
docs/features/<feature-slug>-handoff.md.

This session won't have your project's actual charter/decisions in
context unless you paste them in too — for anything but a trivial
feature, paste the real docs/charter.md and a list of existing
docs/decisions/D-*.md titles (or the full files, if they're not too many)
so section 3/4 below can cite real lines instead of guessing.
-->

# Prompt: draft a spine feature handoff

```
I'm about to hand off a big feature or change for an existing project
that already runs on "spine," an AI-development workflow with a fixed
planning vocabulary. I've pasted the project's docs/charter.md and its
existing docs/decisions/ below (or a summary of them). Interview me about
the feature, then draft docs/features/<feature-slug>-handoff.md in the
exact structure below.

Required sections, in this order:

1. **Feature summary** — one paragraph: what this is, why now. Assume the
   reader already knows the product — don't re-explain it.

2. **Routing** — the single most important section. Based on everything
   else below, decide which ONE of these four applies, and say why:
   - Architecture-shaped (revises or extends a foundational decision —
     state management, persistence, module boundaries, error handling,
     auth model, repo topology) → route to `/design --handoff <path>`
   - Known milestone shape already (no open architecture question, no
     real unknowns beyond implementation detail) → route to `/roadmap`
     via docs/vision.md
   - Genuinely foggy (destination known, path isn't, bigger than one
     sitting) → route to `/wayfinder`
   - Actually task-sized on reflection → route to `/task` directly, and
     say this document wasn't needed
   If genuinely unsure between two, default to the foggier one and say
   so — a redirect from there is cheap; committing to a milestone list
   that hides a real architecture question is not.

3. **Fit within the existing charter** — quote or closely cite the
   specific charter line(s) this feature must respect. If it would
   actually change one, say so explicitly and propose the edit — don't
   let a downstream step infer and apply a charter change silently.

4. **Relationship to existing decisions** — cite every D-<n> this touches,
   extends, or might conflict with, by id, and say for each whether the
   feature fits within it or would need to supersede it. "None — no
   architectural overlap" is a complete answer for a pure feature.

5. **Scope & non-goals** — what this feature deliberately isn't doing,
   even if adjacent and tempting.

6. **Build order** — only if Routing chose the known-milestone-shape
   path. Ordered phases, one-line "why this order" each.

7. **Known unknowns** — only if Routing chose the foggy path. Same three
   buckets as a product spec's own (grilling/prototype/research — ask me
   enough to sort real items into each):
   - Needs a conversation
   - Needs a prototype (real visual/behavioral unknown)
   - Needs research (factual question about existing code/behavior)

8. **Source materials** — real paths or documents this was distilled
   from, or "none" if written fresh.

Only include sections 6 or 7, never both, based on what Routing (§2)
actually decided — don't pad the document with an unused section.
```
