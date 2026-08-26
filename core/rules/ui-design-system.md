---
paths:
  - "**/components/**"
  - "**/screens/**"
  - "**/views/**"
  - "docs/ui/**"
---

# UI design-system discipline

This rule loads whenever a file under a `components/`-, `screens/`-, or
`views/`-named directory (or `docs/ui/` itself) is read. Like
`core/rules/migrations.md` and `core/rules/auth.md`, it names no specific
design tool, token format, or component framework — that specificity lives
in this project's own `docs/ui/` bundle (`core/templates/
ui-handoff.md`) and, where one exists, its `ui-render` adapter
(`core/ADAPTER-CONTRACT.md` §3.3). This rule is the always-applicable
discipline around consuming that bundle, not a replacement for it.

**If `docs/ui/handoff.md` doesn't exist in this project, this rule is a
no-op** — nothing requires a UI handoff bundle to exist, and nothing
below applies until one does.

**Read before you write.** Before touching a UI file, read `docs/ui/
tokens.json` and `docs/ui/components.md`, and — if this change
implements or modifies a screen that already has one — its `docs/ui/
screens/<screen-id>.json` **and the screenshot(s) it points at** under
`docs/ui/screenshots/`. The JSON spec tells you precisely what to use
(which tokens, which components, which props); the screenshot tells you
what the result should actually look like once composed — text alone
routinely underspecifies real visual composition (exact spacing rhythm,
alignment, what "reads right"), which is exactly what the screenshot is
for. Use both, not either. A UI handoff that exists but never gets
read is worse than no handoff at all: it creates the appearance of a
design system nobody's actually building against.

**Use what's declared, don't invent alongside it.** A raw hex color, an
inline magic-number spacing value, or a new one-off component where an
existing library component already covers the case, is exactly the kind of
small, compounding drift `docs/tradeoffs.md`'s duplication concerns already
describe, applied to the visual layer instead of the code layer. Reach for
`docs/ui/tokens.json` and `docs/ui/components.md` first, every
time.

**A real gap is a deviation, not a silent workaround.** If a screen
genuinely needs a token or component the handoff doesn't declare, that's
real information — treat it exactly the way `/task`'s implement phase
already treats any other real surprise: a small, self-contained gap gets
noted (add a row to `docs/ui/handoff.md`'s "Known gaps between the
handoff and the real product") and implementation continues; something
that would change the visual system itself (a new color that isn't a
one-off, a component that should really be added to the library) is a real
deviation — stop and ask, the same halt-tier judgment call any other
class-appropriate surprise gets, rather than inventing new visual language
unilaterally.

**Keep the handoff's screen index honest.** Once a screen from
`docs/ui/screens/<screen-id>.json` is actually built, update its row in
`docs/ui/handoff.md`'s "Screen index" table with the real path(s) it
lives at — the same append-as-you-go discipline a decision record's own
"Implementing paths" section already uses. A screen index that never gets
filled in stops being a map anyone can trust.

**This rule is about grounding, not verification.** It governs what an
agent reads and cites while writing UI code — it does not, on its own,
gate `/verify` or `/ship`. The real automated check is a separate,
dedicated capability, `ui-conformance` (`core/ADAPTER-CONTRACT.md`
§3.9), which runs at verify time for any task that touches a declared UI
path (`core/skills/verify/SKILL.md` §1d) whenever a project has
implemented it — it renders each in-scope screen for real and checks,
deterministically, that the components and tokens declared in its
`screens/<id>.json` actually showed up. This rule is what makes that
check's premise true in the first place — an agent that actually read and
used the handoff while writing the code — not a substitute for having it
run. `ui-conformance` never compares against a screenshot (see
`core/templates/ui-handoff.md`'s own header for why); the screenshot's
role stays entirely upstream, in this rule's "Read before you write" step.
