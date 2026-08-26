<!--
One file per project, at docs/ui/handoff.md — the index for a UI handoff
bundle: tokens, a component library reference, and per-screen UI specs,
in docs/ui/. This is the visual-system counterpart to
docs/design-summary.md (which covers the six architecture categories);
they're kept as separate documents on purpose so each stays legible on
its own — "what did we decide architecturally" and "what does this
actually look like" are different questions with different audiences.

Where this comes from, concretely: build (or have Claude Design build) the
full app design first, then generate a design system/component library and
token set from it, then export a JSON UI description and a screenshot per
screen. Save that bundle here. Nothing downstream requires that exact
workflow, though — any process that produces tokens + a component
reference + per-screen specs (+ screenshots) in this shape works.
`core/prompt-templates/ui-handoff.md` has ready-to-paste prompts for
generating each piece, including one aimed at Claude Design specifically.

What actually consumes this, and how the two halves of the bundle divide
labor:

- **`core/rules/ui-design-system.md`** loads automatically whenever a task
  touches a file under this project's declared UI paths
  (`.spine/ui-paths.conf`) and tells the agent to read `tokens.json`,
  `components.md`, and the relevant screen's `.json` **and its
  screenshot** before writing any UI code. The JSON tells the agent
  precisely *what* to use (which tokens, which components); the
  screenshot tells it what the result should actually look like once
  composed — text alone underspecifies real visual composition, the same
  reason a Figma dev-mode handoff always ships both an inspector and the
  actual frame. Use what's declared, don't invent one-off values, and
  treat a real gap as a deviation worth noting, not a silent workaround.
- **The `ui-conformance` capability** (`core/ADAPTER-CONTRACT.md`
  §3.9), where a project has one implemented, runs at `/verify` time on
  any task that touched a declared UI path: it renders each `screens/
  <id>.json`'s route for real and checks, deterministically, that every
  component and token it declares actually shows up in the render. This
  is a **structural/identity check, not a visual-similarity check** — it
  never compares against the screenshot. Pixel- or perceptual-diffing a
  real render against a golden screenshot is flaky across font
  rendering, anti-aliasing, and dynamic content without dedicated
  image-diff infrastructure spine's core doesn't ship, and a flaky
  floor-adjacent gate erodes trust in every other gate next to it — so
  the deterministic check stays narrowly scoped to what JSON can settle
  unambiguously, and the screenshot's job stays upstream, as grounding a
  human and an agent both look at, never as a blocking gate's input.

Regenerate this index (never hand-patch it into disagreement with the
bundle) whenever the underlying design changes meaningfully — a stale
index that still lists a screen's old route or a component's old name is
worse than an honestly incomplete one.
-->

# UI handoff: `<project-name>`

Last updated: `<yyyy-mm-dd>`
Source: `<where this was generated — e.g. "Claude Design session, <date>">`

## Bundle contents

<!-- What's actually in docs/ui/ right now — keep this list accurate,
     it's the map a human or agent uses to find things without globbing. -->

- `docs/ui/tokens.json` — design tokens (see "Tokens" below)
- `docs/ui/components.md` — component library reference (see
  "Components" below)
- `docs/ui/screens/*.json` — one file per screen (see "Screens" below)
- `docs/ui/screenshots/*.png` — one image per screen state (see
  "Screenshots" below)

## Tokens — `docs/ui/tokens.json`

<!-- The portable, stack-blind source of truth for design values. A
     project's own build tooling translates this into its real format
     (CSS custom properties, a Tailwind config, a theme object) — that
     translation is implementation, not part of this handoff. Illustrative
     shape, not a schema spine enforces — keep whatever categories your
     actual design system has; consistency within the file matters more
     than matching this exactly. -->

```json
{
  "color": {
    "background.primary": { "value": "#0B0E14", "description": "App background" },
    "text.primary": { "value": "#F5F6F8" },
    "accent.default": { "value": "#5B8CFF" }
  },
  "spacing": {
    "xs": { "value": "4px" },
    "sm": { "value": "8px" },
    "md": { "value": "16px" }
  },
  "typography": {
    "heading.lg": { "fontFamily": "...", "fontSize": "28px", "fontWeight": 700, "lineHeight": "34px" }
  },
  "radius": { "sm": { "value": "6px" } },
  "shadow": { "card": { "value": "0 1px 2px rgba(0,0,0,.2)" } }
}
```

## Components — `docs/ui/components.md`

<!-- Human-readable, not raw JSON — an agent needs to reason about *when*
     to reach for a component, not just parse its props. One subsection
     per component. "Implementation status" is the one field that changes
     over the project's life — update it as each component actually gets
     built, the same append-as-you-go discipline docs/decisions/D-<n>.md's
     own "Implementing paths" section uses. -->

```markdown
### Button

Purpose: primary call-to-action trigger.
Variants: primary | secondary | destructive | ghost
Props: label, icon?, disabled?, loading?
States: default, hover, active, disabled, loading
Tokens used: color.accent.default, spacing.sm, radius.sm
Implementation status: not yet built | built at `<path>`
```

## Screens — `docs/ui/screens/<screen-id>.json`

<!-- One file per screen, named by a stable screen-id. Illustrative shape
     — again, not an enforced schema. `components_used` should name real
     entries from components.md, not free-text descriptions, so the two
     files stay cross-checkable by a human skimming both. -->

```json
{
  "screen_id": "checkout-review",
  "route": "/checkout/review",
  "description": "Final review before payment submission.",
  "states": ["default", "empty-cart", "error"],
  "components_used": [
    { "component": "Button", "variant": "primary", "props": { "label": "Place order" } },
    { "component": "LineItemCard", "variant": "default" }
  ],
  "content": {},
  "screenshots": {
    "default": "docs/ui/screenshots/checkout-review.png",
    "empty-cart": "docs/ui/screenshots/checkout-review-empty-cart.png",
    "error": "docs/ui/screenshots/checkout-review-error.png"
  }
}
```

## Screenshots — `docs/ui/screenshots/`

<!-- One image per screen state, named `<screen-id>.png` for the default/
     primary state and `<screen-id>-<state>.png` for every other state
     listed in that screen's own `states` array — referenced back from
     the screen's `screenshots` object above so a reader (human or agent)
     never has to guess the mapping. Full-frame exports at a real
     rendered size, not thumbnails or cropped details — the point is to
     show the actual composition, spacing, and hierarchy a JSON spec
     can't fully convey.

     These are read, not diffed: an agent reads the relevant screenshot
     as visual grounding while implementing (core/rules/ui-design-system.md)
     and a human skims it while reviewing a task's plan or briefing.
     Nothing in spine's core pixel- or perceptual-diffs a real render
     against these — see this file's own header for why that's a
     deliberate choice, not a gap. -->

## Screen index

<!-- One row per screen file — the living map from screen-id to real
     implementation. `Implemented at` starts empty; fill it in once a
     task actually builds the screen, same spirit as a decision record's
     "Implementing paths." A screen with no row here yet is simply not
     built — that's an honest, expected state early in a project. -->

| Screen id | Route | Screenshot(s) | Implemented at |
|---|---|---|---|
| `checkout-review` | `/checkout/review` | `screenshots/checkout-review*.png` | |

## Recommended build order

<!-- Not optional guidance — state this as a real sequencing constraint
     wherever this project tracks those (docs/charter.md's "Sequencing
     constraints" if decided at design time, or a feature's own handoff
     doc's "Build order"/"Sequencing constraints" section per
     core/templates/feature-handoff.md, if decided later): the design
     system/component library gets built and marked `built at <path>` in
     components.md above, for every component the currently-in-scope
     screens actually use, *before* any of those screens' own member
     tasks start. Building screens against an unbuilt or partial
     component library is exactly how visual drift compounds — every
     screen that can't find a real Button invents its own, and
     `ui-conformance` (once implemented) has nothing real to check
     each one against yet either. This mirrors /design's own M0 "walking
     skeleton" reasoning (thinnest real path first) applied one layer up,
     at the visual system instead of the architecture. -->

## Known gaps between the handoff and the real product

<!-- Real, disclosed drift: a screen the design covers that turned out
     to need something not in components.md/tokens.json once actually
     built, or a screen built without ever having a spec here. Each
     entry here is what core/rules/ui-design-system.md calls a deviation
     worth noting — record it here rather than letting the handoff bundle
     silently go stale. "None yet" is a complete, correct early answer. -->

-
