<!--
Three prompts, used in sequence, all in Claude Design (or a comparable
visual-design tool with export capability) — or, per the section below,
Figma. Each stage's output becomes real input to the next. The final
stage's output is what gets saved into docs/ui/ — see
core/templates/ui-handoff.md for the exact file shapes this is
targeting; keep this prompt in sync with that template if either one
changes.

Stage 1 and 2 you likely already do close to this shape. Stage 3 is the
one worth pasting verbatim — it's what makes the export land in the exact
schema core/rules/ui-design-system.md and the ui-conformance capability
both expect, instead of a plausible-looking JSON shape that doesn't
actually match.
-->

# Stage 1 — design the app

This isn't a prompt to paste — "design the app" means nothing without your
actual product context. Design the app in Claude Design the way you
normally would, bringing your own PRD, brief, or context. Before moving to
Stage 2, make sure you've covered:

- Every core screen
- Every state each screen can be in (empty, loading, error, populated,
  and any domain-specific state)
- The navigation between them

Prioritize a coherent, consistent visual language over polishing any
single screen in isolation — the next stage extracts a reusable design
system from what you build here, so inconsistency here becomes
inconsistency in every screen that reuses it.

# If Stage 1 happened in Figma instead of Claude Design

Stages 2 and 3 below assume a Claude Design session that still remembers
"what you just built." If Stage 1 was actually done in Figma, there's no
such session — but Figma already has native primitives for a design
system (Variables for tokens, Components for reusable elements), so
Stage 2 stops being *extraction* and becomes *reading what's already
there*. To let Claude read it, connect Claude Code to the file with
Figma's official MCP server.

**Install (remote server, recommended — works from any Figma file/URL,
no need to have the file open):**

```
claude plugin install figma@claude-plugins-official
```

1. Restart Claude Code.
2. Run `/plugin`, go to the **Installed** tab, select `figma`, press Enter.
3. Press Enter again to open the browser auth page, and click "Allow access."
4. Back in the terminal, run `/plugin` again — it should show `figma` as
   **connected**.

**Desktop server instead (org/enterprise-only, requires the file open
locally in Dev Mode):** in the Figma desktop app, open the file, toggle
Dev Mode, enable the MCP server in the right sidebar, and copy the local
server URL it shows. Then in Claude Code:

```
claude mcp add --transport http figma-desktop http://127.0.0.1:3845/mcp
```

Restart Claude Code afterward. Note the desktop server generally acts on
whatever's currently selected on the canvas, so pulling many
screens/states this way is a per-frame loop (select → pull → next) —
prefer the remote server, addressed by node URL, when covering a lot of
screens.

**Stage 2, adapted for Figma** — run this in Claude Code, in this repo,
with the `figma` MCP server connected:

```
Read the design system directly from this Figma file: <file URL>.
Pull every variable/style as a token, and every component's real
variants, props, and states as actually defined on the file — don't
invent or speculate. Flag anything used inconsistently (e.g. two
near-duplicate colors that should be one token), and flag any screen
element that isn't backed by a real component or variable.
```

Reconcile anything flagged — either fix it in the Figma file, or note the
inconsistency knowingly — before moving to Stage 3.

**Stage 3, adapted for Figma** — same target schema as Stage 3 below, but
Claude Code can pull it straight from the file and write the bundle
directly into `docs/ui/`, no manual copy-paste:

```
For every core screen and every state (empty, loading, error, populated,
domain-specific) in this Figma file: <file URL>, pull the token values in
use, the components used with their real variant/props, and an image
export of that screen/state. Write it out as the docs/ui/ bundle in the
exact shape below, saved directly into this repo.
```
followed by the same schema block from Stage 3 below (tokens.json,
components.md, screens/<screen-id>.json, screenshots/<screen-id>[-<state>].png).

Then continue with "After exporting" as normal.

# Stage 2 — extract the design system

```
Now extract a design system from what you just built:

1. A token set covering every real value in use — color, spacing,
   typography, radius, shadow — named semantically (e.g.
   "background.primary", not "gray-900"), not just a literal value dump.
   Consolidate near-duplicate values you used inconsistently (two blues
   that should be one) rather than exporting both.
2. A component library: every reusable UI element you used more than
   once, with its real variants, props, and states as actually used
   across the screens — not a speculative superset of what a component
   library "should" have.

Make sure every screen actually uses these tokens and components — go
back and reconcile any screen that drifted from them during stage 1.
```

# Stage 3 — export the spine-shaped handoff bundle

```
Export what you just built as a file bundle in exactly this shape, so it
can be dropped into a project's docs/ui/ directory unchanged:

docs/ui/tokens.json — one JSON object, grouped by category (color,
spacing, typography, radius, shadow, ...), each leaf an object with at
least a "value" key:

{
  "color": {
    "background.primary": { "value": "#0B0E14", "description": "..." }
  },
  "spacing": { "sm": { "value": "8px" } }
}

docs/ui/components.md — one markdown section per component, this
exact field set, in this order:

### <ComponentName>

Purpose: <one line>
Variants: <variant> | <variant> | ...
Props: <prop>, <prop>?, ...
States: <state>, <state>, ...
Tokens used: <token.path>, <token.path>, ...
Implementation status: not yet built

docs/ui/screens/<screen-id>.json — one file per screen, kebab-case
screen-id, this exact shape:

{
  "screen_id": "<same as filename, no extension>",
  "route": "<intended route path>",
  "description": "<one line>",
  "states": ["default", "..."],
  "components_used": [
    { "component": "<name from components.md>", "variant": "<variant>", "props": { } }
  ],
  "content": { },
  "screenshots": {
    "<state>": "docs/ui/screenshots/<screen-id>-<state>.png"
  }
}

Every "component" name in components_used must exactly match a heading in
components.md — no free-text descriptions in place of a real component
name. If a screen used something that isn't in the component library,
that's real information: either add it to components.md in stage 2 and
re-export, or list it in this screen's JSON with a "not_in_library": true
flag rather than silently naming it as if it were a library component.

docs/ui/screenshots/<screen-id>.png — full-frame export of the
default/primary state, real rendered size, not a thumbnail or cropped
detail.
docs/ui/screenshots/<screen-id>-<state>.png — one per additional
state listed in that screen's own "states" array, same export rules.
Every path named in a screen's own "screenshots" object above must
correspond to a real exported file — don't reference a state's screenshot
that wasn't actually exported.

Give me all of this as a downloadable file tree, not inlined chat text I
have to reassemble by hand.
```

# After exporting

Save the bundle as `docs/ui/{tokens.json, components.md,
screens/*.json, screenshots/*.png}`. Then write `docs/ui/handoff.md`
itself from `core/templates/ui-handoff.md` — its "Bundle contents"
list and "Screen index" table are just real metadata about what you just
saved (screen ids, routes, screenshot paths), quick to fill in by hand or
by asking the same Claude Design session to draft the index too, pointed
at that template.

