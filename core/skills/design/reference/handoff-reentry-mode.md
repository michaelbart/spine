# Design — handoff re-entry mode (loaded on demand)

Loaded from SKILL.md when: `--handoff <path>` was given and `docs/decisions/D-*.md` already existed at §0.

**Only when `--handoff <path>` was given and `docs/decisions/D-*.md`
already existed at §0.** This replaces §§1–4 and §7 for this run; every
other section (§5, §6, §7.5, §7.6, §8) still applies, scoped as noted
inline below and in each of those sections' own text.

**Registry check first, before proposing anything.** Look for
`.spine/hooks/design-registry-diff` (a fixed, conventional path — no
manifest entry, this is not a capability adapter and is never floor-gated
or self-test-required; it's a lighter, opt-in, `/design`-only check).

- **If it exists**, run it: `.spine/hooks/design-registry-diff <handoff
  path> --project <project root>`. Non-zero exit: show its diagnostics
  verbatim to the human and stop — don't propose a single decision until
  this is resolved, the same "collision blocks everything downstream"
  posture a real id collision deserves. Zero exit: continue. Say it in this
  form (per `core/templates/human-touchpoint.md`):

  <!-- touchpoint:start short -->
  > **What happened:** the design registry check found a conflict between this handoff and what's already recorded: <the script's message, quoted>.
  > **What it means for you:** I've made no design decisions yet, because anything I proposed could clash with the existing record.
  > **To continue:** resolve the conflict quoted above (edit the handoff or the registry entry it names), then tell me to re-run the check.
  <!-- touchpoint:end -->
- **If it doesn't exist**, offer to draft one — this project's own
  handoff shape and canonical registry files are never something spine
  itself knows (it has no vocabulary for "screen" or "component"; that's
  entirely this project's convention), so there is no generic template to
  fall back on. If the human agrees: read the actual handoff and whatever
  canonical registry files it references (asking where they live if not
  obvious), infer the real id pattern and collision logic for *this*
  project, and draft a script following the same contract every spine
  script uses — quiet one-line output on success, exit non-zero with one
  diagnostic line per collision on failure (`core/scripts/next-milestone-
  task`'s own header is a good model of that contract, even though this
  script's content is unrelated). Show the draft before saving. On
  approval, write it to `.spine/hooks/design-registry-diff`, `chmod +x`,
  then run it for real per the bullet above. If the human declines
  (either to draft one, or to run an existing one this time), proceed
  without the check and say so plainly when presenting decisions below —
  never silently skip it and let the omission read as "checked, clean."

**Classify the handoff's content, don't re-interview.** For each real
piece of scope the handoff introduces, decide against the same six
categories §1 uses: does it require *revising* an existing adopted
decision (propose superseding it — append-only, same as §6's own
"Revise" outcome, human confirms), does it need a *new* decision (write
`D-<n>` the normal way, §1's own template/fields, scoped to only what
this handoff actually needs — not a full six-category re-walk), or does
it carry no architectural weight at all (no decision — this should be the
common case; most of a UI handoff is pure content, not architecture).
Never manufacture a decision because content arrived; the "eager
architect" caution at the top of this file applies here exactly as it
does on a first run.

**The reconciliation rule.** If the handoff states its own "mechanism
wins here, design wins there" note (or any comparable resolution of a
tension it's aware of), record it as a `Leaves open:` line in the
relevant decision's own `## Consequences` — never in `work/M<n>/
milestone.md`'s intro prose, which this skill doesn't own past M0. This
is the one deliberate design choice that makes the rest of the loop work
without any new absorption mechanism: `/roadmap`'s existing decision-
follow-ons step (§1b there, unchanged) already knows how to read a
`Leaves open:` line and sequence it into the right milestone.

**Converge at §5**, scoped automatically: every decision this section
wrote is `proposed`, every decision from a prior session is already
`adopted`, and §5/§6's own "every `docs/decisions/D-*.md` currently
`proposed`" instruction already means exactly this pass's work — no
separate scoping edit needed there. **One path change carries through
§5/§6 in this mode**: write to `work/design/design-review-<handoff
basename>.md` and `work/design/artifacts/<agent>-verdict-<handoff
basename>-raw.json`/`-<handoff basename>.json` instead of the fixed
un-suffixed paths §5 names — a first design session only ever ran once,
so those paths being fixed was never a collision risk before; a second
handoff pass reusing them would silently overwrite the original session's
review record (or an earlier handoff pass's) rather than adding to the
project's history.

**This mode's own completion check, in place of §7.** Don't run
`design-gate` — its cross-category coverage, decision cap, and M0
capability-targets check are calibrated for a first run and don't apply
to a scoped follow-up. This pass is done when every kept verdict from §6
has a resolved outcome (revise or recorded override) — nothing more.

**Stamp the handoff as consumed** — write `.spine/handoff-consumed-sha`:
line 1 the handoff path exactly as passed to `--handoff`, line 2 `git
hash-object <path>`'s output. This is what lets `/spine` later notice if
this same document changes again without going through `/design
--handoff` a second time (`core/skills/spine/SKILL.md`'s own staleness
check) — overwrite any prior stamp for this same path; a stamp for a
*different* path (a second, distinct handoff document) is a separate
concern §0's own product-spec.md check would have already surfaced, not
something this line silently loses.

**§8, this mode's own ending.** `git add -- docs/decisions/
docs/design-summary.md .spine/handoff-consumed-sha` (never `work/M0/`,
`.spine/capabilities.json`, or `.spine/adapters/` — this mode doesn't
touch any of them) plus `.spine/hooks/design-registry-diff` if this run
created or updated it. Commit message: `"spine: design handoff <handoff
basename> — <n> decisions added, <m> revised"`. Tell the human how many
of each, what (if anything) got deferred or overridden, and the next
command: if what's left to plan is a known milestone list, **`/roadmap`**;
if the remaining shape is still genuinely foggy (this handoff opened up
more than it closed), **`/wayfinder`** first — either way, not `/task
--milestone M0`. The new `Leaves open:` lines are real citable follow-ons
now, and both `/roadmap`'s absorption step and a `/wayfinder` map seeded
from them pick them up without any change on their end.
