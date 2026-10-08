# Proposal: `ui-fidelity` — a real visual-fidelity check in `/verify`

Status: core side implemented (scripts, agent, contract, skills, templates,
self-tests); no project `ui-capture` adapter exists yet, so nothing has run
end to end. Original design text follows; deviations: `adversary-cache-tier`
needed no change (it is agent-agnostic), and the content-provenance work
(`content-sources-check`, `.spine/ui-content-paths.conf`) was added — see
`docs/tradeoffs.md`. Written 2026-09-23 from a read of
`core/ADAPTER-CONTRACT.md` (§1, §3.3, §3.9, §4, §5, §5.1),
`core/skills/verify/SKILL.md` (§1c, §1d, §3, §5), `core/agents/{falsifier,security}.md`,
`core/scripts/{verdict-filter,ui-touch}`, and turnpilot's
`.spine/adapters/{ui-render,ui-conformance}` + `docs/ui/screens/make-ready-board.json`.

## The gap, restated against what the code actually does

`ui-conformance` (turnpilot `.spine/adapters/ui-conformance`) does
`html.includes('data-component="Name"')` per declared component plus two
token color strings. By construction it cannot see layout, variant/fill,
spacing, density, or a component's own internal rendering. `ADAPTER-CONTRACT.md`
§3.9 says so on purpose ("never the input to this capability's pass/fail
decision"), and `ui-design-system.md` / `ui-handoff.md` repeat it. Nothing
in spine reads `docs/ui/screenshots/*.png` after the agent's own
write-time grounding step. That is the hole 20260923-home-make-ready-board
fell through.

> Update: turnpilot is building a signed-in Playwright context and a
> dev-only state gallery (`/__ui/<screen>?state=<state>`). `ui-capture`
> consumes those instead of its own auth/step files where they exist; see
> contract §3.10 "Authentication and state driving". The text-provenance
> check is reviewer mandate 6 plus the plan's `## Content sources` gate.

## Facts that constrain the design

1. **The unit is a state, not a screen.** Each `screens/<id>.json` lists
   `states[]` and a `screenshots` map with one PNG per state (make-ready-board:
   12; home: 13 states but only 11 PNGs). There is one spec JSON per screen,
   not per state. **v1 compares every state that has a screenshot.** The
   spec says nothing about *how to reach* a state (open a popover, select
   rows), and no adapter drives interactions today, so v1 adds a state-driver
   file (below). A state listed in `states[]` with no screenshot, or with a
   screenshot but no driver, is reported by name as `not compared` with the
   reason, never silently dropped (home's `consent-request`, `consent-modal`,
   `property-budget-drawer` mismatch is the live example).
2. **Reference and render differ in size** (reference 1675x1242 for
   make-ready-board, 1679x1239 for home). Render at the reference's own pixel
   dimensions (read from the PNG header) so a reviewer sees like-for-like.
3. **The spine repo owns contract/skills/agents/scripts; adapters live in each
   project.** So this splits across two places (see "Where the pieces live").
4. **`ui-built-screens.txt` is not in core at all.** It is a turnpilot-local
   convention that `ui-render` and `ui-conformance` each re-implement. The
   requirement "scoped to built screens" is therefore only as good as each
   adapter's copy. This change should promote it into the contract (see below)
   rather than copy it a third time.
5. **`verdict-filter` is agent-agnostic** (reads `.agent` from the file), but
   `/ship` §3a gathers only `falsifier` and `security` verdict files, and
   `adversary-cache-tier` is keyed on agent name. Both need to know about a
   new agent.
6. **Evidence kinds are `file_line | command | decision`.** None fits "the
   render shows X where the mockup shows Y." Left as-is, a fidelity reviewer's
   every finding would be dropped by `verdict-filter` for malformed evidence.

## Decision: reviewer agent, fed by a scripted capture adapter

Not either/or. The judgment cannot be a script; the *inputs* to it can and
should be.

**Why not a scripted adapter alone.** §4's self-test convention needs a
fixture guaranteed to pass and one guaranteed to fail, routed through the
real check. "Does this layout match this mockup" has no such fixtures without
either pixel diffing (§3.3/§3.9 reject it on flakiness grounds, and it would
also flag every legitimate difference in seeded data) or an ML classifier we
don't ship. Building a script that fakes a boolean here would give exactly the
false confidence `ui-conformance` gave.

**Why a reviewer agent.** It is the same shape as `falsifier`/`security`: a
fresh-context subagent given real artifacts and no implementer narrative, asked
for concrete discrepancies with evidence, filtered mechanically by
`verdict-filter` so "a script decides what counts as evidence, never a model"
(§5) still holds. Multimodal comparison of two images against a spec is what
this class of agent is for.

**Why also a scripted piece.** A reviewer looking at two PNGs alone will
produce vague or hallucinated findings ("spacing looks tighter"). The four
real misses were all *measurable* from the DOM once you have it: a
two-column split is bounding boxes; filled-vs-outlined is computed
`background-color`/`border`; run-together text is text nodes with abutting
rects; an unused `badge` prop is a declared prop with no rendered node. So a
deterministic capture step emits those facts, and the reviewer must anchor
every finding to one.

### Component 1 — `ui-capture` adapter (new capability #20, per project)

- Sits in §3's "operates on nothing" row, like `ui-render`/`ui-conformance`.
  Exit 0 = every in-scope screen was captured; non-zero = a capture failed
  (route 404s, no body, browser error). **It never judges fidelity.**
- **State driving.** New per-screen file `docs/ui/states/<id>.json`
  mapping each state name to an ordered list of steps Playwright runs
  from a fresh load of the route: `click`/`fill`/`press` addressed by
  role+name or visible text (not CSS selectors or product-side test hooks),
  plus an optional `wait_for` text. `default` needs no entry. Chosen over a
  `?state=` query hook because it needs no product code, and over inferring
  steps from state names because that would be guessing. Authoring cost falls
  only on built screens (scoping below). Capture fails soft per state: a
  step that can't find its target marks that state `driver_failed` with the
  locator error (a real finding for the task, distinct from `no_driver`).
- Output directory via an env var, following the `SPINE_BASE_REF` /
  `SPINE_CONTRACT_NAME` precedent (§3.2), not a positional arg:
  `SPINE_UI_CAPTURE_DIR`. Per built screen `<id>` and per state `<state>` it writes:
  - `<id>/<state>.render.png` — screenshot at that reference PNG's dimensions
  - `<id>/<state>.reference.png` — copy of `screenshots[<state>]`
  - `<id>/<state>.facts.json` — for each `data-component` element: name, `data-variant`
    if present, bounding rect, computed `background-color`, `color`,
    `border`, `font-size/weight`, visible text; plus the viewport size and
    the list of declared `components_used[]` entries (with props) that had **no**
    rendered node.
  - `<id>/spec.json` — the screen's `docs/ui/screens/<id>.json`, verbatim
  - `<id>/coverage.json` — every state in `states[]` with status
    `captured | no_screenshot | no_driver | driver_failed`
- Reuses `ui-render`/`ui-conformance`'s Playwright + dev-server bring-up
  (the same duplicated code; this is the third copy — extract it to a shared
  `_ui-lib.sh` beside `_smoke-lib.sh` as part of the turnpilot side of this
  work, otherwise the three drift).
- **Scoping**: reads `.spine/ui-built-screens.txt` exactly as the other two do.
  Absent/empty behaves as they do today (every screen). A screen or state
  with nothing to compare to is recorded in `coverage.json` with its reason,
  not treated as a capture failure; `driver_failed` is.
- **Self-test** (§4): `pass` builds a throwaway page with two marked
  components + a throwaway 1x1-PNG reference per state and spec, with one non-default
  state reached by a click step; runs the *real* capture path; asserts PNG
  magic bytes, non-empty `facts.json` per state containing both components with
  non-zero rects, and that the clicked state's facts differ from default's.
  `fail` serves a page whose state step targets a control that doesn't exist
  (must exit non-zero with `driver_failed`), and separately an empty body.
  This is fully deterministic — it proves the capture works, not that the UI is
  right, which is the honest scope.

### Component 2 — `ui-fidelity` agent (new, in core)

`core/agents/ui-fidelity.md`. Tools: `Read, Grep, Glob` (Read is how it views
PNGs). Read-only, no worktree, no `Bash` (nothing to run; capture already
happened). `model: inherit`, `effort: high`. Delegation message gives exactly:
the capture dir path and the screen ids. Not the plan, not the diff, not the
implementer's narrative — the same independence rule.

One agent dispatch **per screen** (all its states inside one run, since
the states share components and the reviewer should notice cross-state
inconsistency); screens dispatch in parallel. Mandate, per state, in order:
1. **Layout structure** — column/region split, ordering, relative widths,
   from the two images, corroborated by `facts.json` rects.
2. **Variant/emphasis per documented row** — for each `components_used` entry,
   does the render's fill/border/weight match the reference for that element
   (filled primary vs outlined, danger vs neutral).
3. **Component integrity** — a component's own content rendering (props run
   together, clipped, overlapping), from `facts.json` text/rects.
4. **Declared-but-unused affordances** — props in the spec absent from render.
5. **Spacing/density/hierarchy** — only when a measurable delta exists
   (e.g. row height, gutter width in px from facts vs. measured off the
   reference); otherwise not a finding.
6. **Content** — literal strings in `spec.content` absent from the render,
   with dynamic data (counts, names from seed) reported `low` at most.

It must **not** flag: font antialiasing, exact pixel offsets under a
stated tolerance (a few px), seed-data differences, or one state's screenshot
being used to judge another. Each finding names its state. The reply lists
`attacked` per screen and state, including the explicit "states not compared:
[...] (reason)" line taken from `coverage.json`, so the clean-bill rule (`attacked`
non-empty) is meaningful and coverage limits are visible.

Reply is the standard §5 JSON, `agent: "ui-fidelity"`, so §5.1's shared
discipline applies by pointer, not by copying.

### Component 3 — a new evidence kind: `render`

Add to §5 and `verdict-filter`:

```json
{ "kind": "render", "screen_id": "make-ready-board", "state": "sort-open",
  "artifact": "make-ready-board/sort-open.facts.json",
  "quote": "\"component\":\"ContextualAction\",...\"backgroundColor\":\"rgba(0, 0, 0, 0)\"" }
```

`verdict-filter` gets a pass-2 check exactly like `decision`'s: `artifact` must
resolve to a real file in the capture dir and `quote` must appear verbatim in
it. A model can still misread what a fact means, but it cannot cite a fact that
isn't in the capture. That preserves "a script decides what counts as
evidence." Pass 1 shape check: non-empty `screen_id`, `artifact`, `quote`.
A finding with no measurable anchor (pure "looks different") has no valid
evidence shape and is therefore **dropped**. A finding that cannot cite a
captured fact is not reportable. This deliberately trades recall for
trust, matching the "flaky gates erode trust" thesis; the calibration below
measures whether that trade misses too much.

### Component 4 — `/verify` step 1e and eligibility

New step after 1d, same shape, own key, own section:

- Reuses step 1c's `ui-touch` result (never re-run); `ui_touched: false`
  everywhere → omit whole section.
- Class eligibility: `jq -r '.ui_fidelity_class1_optin // false' ~/.spine/user-config.json`.
  Class 2 always eligible; Class 1 only if true; else the section reads
  `SKIPPED (Class 1, ui_fidelity_class1_optin not set)`. Separate key from
  the other two for the reason §3.9 already gives (this is the costliest and
  most opinionated of the three: a multimodal agent run).
- Eligible: check `ui-capture` in `.spine/capabilities.json`. Not
  `implemented` → degraded with its recorded reason (the common case for a
  project with no `docs/ui/screenshots`), listed under Capability gaps. Never
  silent.
- `implemented` → run `.spine/adapters/ui-capture` (not through `floor`) with
  `SPINE_UI_CAPTURE_DIR=work/<task-id>/artifacts/ui-fidelity/`, then
  dispatch the `ui-fidelity` agent, then `verdict-filter` →
  `work/<task-id>/artifacts/ui-fidelity-verdict.json`.
- Capture **failure** is a real failure of step 1e's precondition: recorded
  in the section; it does not become a passing "no findings."
- Budget: per screen agent, scaled by state count (~1 min/state, floor 5 min, cap 20), cancel-and-record on overrun
  (same rule as §3's adversaries).
- Adversary cache tier: content-hash key includes the screen's `.tsx` blast
  radius **and** the reference PNG and spec JSON hashes, so a changed mockup
  invalidates a prior clean pass.

### Blocking or not (decided: human disposition required)

**Findings do not change `/verify`'s PASS/FAIL**, same as every other
adversary finding (§6 of verify) — a model judgment must never be a floor
gate. A **capture failure** does fail step 1e's line, like a `ui-render`
failure does. The teeth come from the second half: extend `/ship` §3a
(flagged-finding triage) to gather `ui-fidelity-verdict.json` alongside
falsifier/security, so every kept `not_fixed` finding needs an explicit
human disposition (fix now, or carried into `milestone.md` Known gaps) before
ship, and a `driver_failed` state is surfaced there too. This is the decided
gating. That closes the "shipped-to-verify and nobody looked" hole without
making a model the gatekeeper. Open question for the user, below.

## Changes required (core repo)

| File | Change |
|---|---|
| `core/ADAPTER-CONTRACT.md` | §1: 19 → 20 capabilities, add `ui-capture` prose; new §3.10; §3.3/§3.9 "never against a screenshot" wording narrowed to "never a *pixel/perceptual* gate" and cross-referenced; §5: `render` evidence kind + `ui-fidelity` agent; promote `.spine/ui-built-screens.txt` into §3.3 as a contract convention |
| `core/agents/ui-fidelity.md` | new |
| `core/skills/verify/SKILL.md` | step 1e, §2 adversary-count note (`ui-fidelity` is not counted in `class1_adversaries`), §5 "UI fidelity" section, §6 |
| `core/templates/verify.md` | new "UI fidelity" section |
| `core/scripts/verdict-filter` | `render` kind, pass 1 + pass 2 |
| `core/scripts/adversary-cache-tier` | accept `ui-fidelity` |
| `core/skills/ship/SKILL.md` §3a | gather `ui-fidelity-verdict.json` |
| `core/templates/ui-handoff.md` | document `docs/ui/states/<id>.json` (state drivers) beside screens/screenshots |
| `core/scripts/core-selftest` | verdict-filter `render` cases: valid, fabricated quote, missing artifact, malformed shape |
| `core/rules/ui-design-system.md`, `core/templates/ui-handoff.md`, `README.md` | replace "never compares against a screenshot" with the accurate split: conformance = structural, fidelity = reviewed, neither is pixel-diff |
| `core/skills/{bootstrap,adopt}/SKILL.md` | `ui-capture` calibration: implemented iff `ui-render` is *and* `docs/ui/screenshots/` exists; new opt-in key added to the recalibration prompts |
| `docs/tradeoffs.md` | new section (why agent+capture, why `render` evidence, why non-blocking, recall/trust tradeoff) |

**Merge hazard**: `ADAPTER-CONTRACT.md`, `core-selftest`, `floor`,
`design/SKILL.md`, `ship/SKILL.md`, and `docs/tradeoffs.md` all have
uncommitted edits in the working tree right now. Implementation should start
from those being committed or stashed, not layered on top.

## Changes required (turnpilot, separate repo, separate task)

`.spine/adapters/ui-capture` per above; extract shared Playwright/dev-server
bring-up out of `ui-render`/`ui-conformance` into a shared lib; register
`ui-capture` in `.spine/capabilities.json`; set `ui_fidelity_class1_optin` in
`~/.spine/user-config.json` if wanted on Class 1 (it currently has the other two
set `true`; the absent-key default is `false`, so nothing turns on unasked).

## Acceptance for the implementation task

1. `core-selftest` green including the new `render` evidence cases;
   `adapter-conformance` green for `ui-capture` in turnpilot (both self-test
   modes, exercising the real capture path).
2. **Calibration, not a unit test — this is the only real evidence the
   reviewer works.** Run capture + agent on the two 20260923 screens *as they
   were before the fixes* (rebuild from the pre-fix commit). Pass = it reports,
   with valid `render` evidence, at least: the single-column layout, the
   outlined-vs-filled ContextualAction, the CapacityVerdict run-together text,
   and the unused NavRailItem badge. Then run it on the fixed screens and
   record the false-positive count. Result goes in `docs/tradeoffs.md`. If it
   catches fewer than all four, that is a finding about the design, not a
   reason to loosen the acceptance bar.
3. Calibration also covers non-default states: known defects are checked in
   every state where they're visible, and every declared state ends up
   `captured` or listed not-compared with a reason.
4. Ineligible/skipped/degraded paths each produce their distinct
   `verify.md` wording (Class 1 unopted, capability not implemented, no
   `ui_touched`), verified by reading a generated `verify.md` for each.

## Known limits (disclosed, not hidden)

- A state is compared only if it has both a screenshot and a driver entry;
  the rest are listed by name and reason each run. Driver steps are
  hand-authored per built screen and can go stale when the UI's labels change
  (surfaced as `driver_failed`, not as a pass).
- Overlays (popovers, drawers) shift bounding-box facts; the reviewer must
  judge those states against their own reference, never against `default`.
- The reviewer's verdicts are a model's judgment. `render` evidence proves the
  cited fact exists in the capture; it does not prove the model drew the right
  conclusion from it. Expect some false positives; the calibration measures it.
- Recall is deliberately capped by "must cite a captured fact": a visual
  difference that no captured measurement expresses will not be reported.
- Cost: one image-reading agent run per UI-touching Class 2 task, and per
  Class 1 task only for users who opt in.

## Open questions (my recommendation first)

1. ~~Ship gating~~ — decided: non-blocking at `/verify`, mandatory human
   disposition at `/ship` §3a.
2. ~~State coverage~~ — decided: all states in v1, via `docs/ui/states/<id>.json`
   step files. Remaining question: is role/text-addressed click steps
   acceptable, or would you rather add a `?state=<name>` hook in apps/web?
   I recommend step files (no product code, no test-only paths shipped).
