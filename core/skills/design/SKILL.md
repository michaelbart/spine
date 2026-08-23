---
name: design
description: The design stage — a facilitated, interactive session that turns a confirmed charter into foundational decisions, a walking-skeleton milestone, and a reviewed stopping point before any feature code exists. Run after /bootstrap on greenfield; may also run on brownfield to make implicit architecture explicit (not exercised by this build). An optional --handoff feeds it an external design document — a product spec as extra grounding on a first run, or a supplementary design delivery to reconcile against decisions that already exist.
disable-model-invocation: true
argument-hint: [--project <path>] [--handoff <path>]
---

You are running `/design` against `--project <path>` from `$ARGUMENTS`
(default: current directory), plus an optional `--handoff <path>` (an
external design document — what it means depends on whether decisions
already exist, §0 below). Scripts at
`${CLAUDE_SKILL_DIR}/../../scripts/<name>`, templates at
`${CLAUDE_SKILL_DIR}/../../templates/<name>`, agents `falsifier`/`security`
in design mode (`core/agents/falsifier.md`/`security.md` §"Design-mode
mandate"). Hand `${CLAUDE_SKILL_DIR}/../../...` to the shell verbatim,
`../../` included — do **not** lexically collapse it to `.claude/`;
`.claude/skills/design` is a symlink into the spine core checkout, and
collapsing the text yields a nonexistent `.claude/scripts/...` path.

**This is a facilitated conversation, not a batch generation.** The human
is present for every foundational category below — you propose, they
decide. Do not write a decision record the human hasn't actually confirmed
just because a plausible answer occurred to you; that is exactly the
"eager architect" failure mode (more decisions reads as more thorough, it
isn't) this skill exists to avoid.

## 0. Preflight

`.spine/capabilities.json` and `docs/charter.md` must both already exist
(from `/bootstrap`) — if either is missing, stop and say so; `/design`
is not an install mechanism. If `docs/charter.md` still reads `DRAFT`, tell
the human plainly that grounding decisions on an unconfirmed charter means
those decisions may need revisiting the moment the charter is — ask
whether to pause here and confirm the charter first (recommended) or
proceed anyway with that risk stated. Don't silently proceed as if the
charter were final.

If `docs/decisions/D-*.md` already exist, this is a resumed or re-run
design session, not a first one — say so, show what's already decided, and
treat each category below as "confirm or revise," not a cold re-interview.

**`--handoff <path>` branches on this same check.** If given and
`docs/decisions/D-*.md` do **not** yet exist, this is a first run with
extra grounding — read the handoff now alongside the charter, per §1's own
note, and continue through §§1–8 exactly as below. If given and decisions
**do** already exist, skip §§1–4 and §7 entirely — go to **"Handoff
re-entry mode"** below instead, then converge at §5. (`--handoff` with no
prior decisions and no charter yet is the one combination that can't
happen — `/design` still requires `docs/charter.md` above regardless.)

## 1. Walk the six foundational categories

**(Skip this section through §4 entirely in handoff re-entry mode — see
"Handoff re-entry mode" below, then resume at §5.)**

**If `--handoff <path>` was given on a first run** (§0), read it now,
alongside the charter, before proposing anything below — a product spec or
an external design document genuinely bears on these categories (a
real-time collaborative feature says something about state-management and
persistence a charter alone won't spell out). Cite it the same way you'd
cite a charter line when it's what a proposed answer actually follows
from. This doesn't add a seventh category or change what counts as
grounding for the charter DRAFT check above — it's additional context for
the same six questions, nothing more.

For each of **state-management, persistence, module-boundaries,
error-handling, auth-model, repo-topology** (the same fixed six
`core/scripts/design-gate` checks — don't invent a seventh or rename one):
propose a concrete answer grounded in the charter (cite the charter line
it follows from), let the human confirm, revise, or defer it. A category
doesn't have to produce a decision — deferring it to `docs/decisions/
DEFERRED.md` (§3) is a legitimate, first-class outcome, not a fallback for
running out of time.

For each confirmed category, write `docs/decisions/D-<n>-<kebab-slug>.md`
from `${CLAUDE_SKILL_DIR}/../../templates/decision.md` (next id: one more
than the highest existing `docs/decisions/D-<n>-*.md`, or `1` if none
exist). Fill every required field: `Status: proposed` (never `adopted`
yet — that only happens after design review, §5), `Category` (exactly one
of the six), `Source: design session`, `Scope` (real globs — this is what
class escalation and future rules will consume, don't leave it vague),
`Alternatives rejected` with real reasons (required here, unlike the
`/ship`-distilled path — if you can't name a real alternative that was
actually weighed, the category probably wasn't decided yet, just asserted).

**Repo topology's decision determines whether this becomes a multi-repo
project** (Extension B territory) — if the human's answer here is "more
than one repository," say so plainly and note that `/workspace` (not this
skill) is what turns that decision into a real workspace; `/design` itself
still finishes this single design session normally.

## 2. Capability planning

The stack is itself a design decision by this point (repo-topology,
possibly module-boundaries, already answered §1) — finalize
`.spine/capabilities.json`'s planned statuses now, regardless of what
`/bootstrap`'s own Layer 3 left them as:

- **`typecheck`, `lint`, `secret-scan`, `dep-diff`** — these don't need
  real application behavior to check, only a real stack and structure,
  which §1 just confirmed. If any of the four is not already `implemented`
  (bootstrap may have left it `not-applicable` when there was nothing
  concrete to point at yet), generate its real adapter now
  (`core/ADAPTER-CONTRACT.md §2–4`) and run
  `${CLAUDE_SKILL_DIR}/../../scripts/adapter-conformance --all --project <project>`
  — do not mark it `implemented` unless conformance passes.
- **Every other capability** (`test`, `test-changed`, `clone-scan`,
  `callers`, `mutate`, `smoke-seed`, `smoke-run`, `smoke-golden`,
  `migrate-rehearse`) — mark `unavailable` with reason exactly `"no code
  yet — skeleton target"` unless a real reason from `/bootstrap` already
  says something more specific (an unavailable tool for this stack, not
  just "no code yet") — don't overwrite a real, specific reason with the
  generic skeleton-target one. These are milestone 0's job to flip, not
  this skill's.

## 3. `docs/decisions/DEFERRED.md`

For every foundational category from §1 that didn't produce a decision,
write an entry from `${CLAUDE_SKILL_DIR}/../../templates/DEFERRED.md`:
`Category` (exact match), `Item` (what's actually left open), `Trigger` (a
concrete, checkable event — "first task needing X decides this," never
"decide later"). **An empty file here, with every category already
decided in §1, is the correct outcome sometimes — don't manufacture a
deferred item to look thorough.** Conversely, a category with neither a
decision nor a real entry here fails the stopping rule (§6) by
construction; that's deliberate, not a bug to work around.

## 4. Milestone 0 — the walking skeleton

Write `work/M0/milestone.md` from
`${CLAUDE_SKILL_DIR}/../../templates/milestone.md`: the thinnest possible
end-to-end path through the topology just decided — real persistence, real
auth if the auth-model decision implies one, nothing else. Member tasks
listed as `TBD` (real task IDs get filled in once `/task --milestone M0`
creates them, §8's handoff, not here). **Capability targets table**:
`test`, `smoke-seed`, `smoke-run`, `smoke-golden`, each with its current
planned status from §2 (`unavailable (no code yet — skeleton target)` for
all four, typically, at this point) — the done-definition names these four
reaching `implemented` as what "done" means; don't write a done-definition
that's vaguer than that.

## Handoff re-entry mode

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
  posture a real id collision deserves. Zero exit: continue.
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

**§8, this mode's own ending.** `git add -- docs/decisions/
docs/design-summary.md` (never `work/M0/`, `.spine/capabilities.json`, or
`.spine/adapters/` — this mode doesn't touch any of them) plus
`.spine/hooks/design-registry-diff` if this run created or updated it.
Commit message: `"spine: design handoff <handoff basename> — <n>
decisions added, <m> revised"`. Tell the human how many of each, what (if
anything) got deferred or overridden, and that **`/roadmap`** — not
`/task --milestone M0` — is the next command: the new `Leaves open:` lines
are real citable follow-ons now, and `/roadmap`'s existing absorption step
picks them up without any change on its end.

## 5. Design review

Run both adversaries in design mode, fresh `Agent` calls
(`subagent_type: falsifier` / `subagent_type: security`), delegation
message pointing at `docs/charter.md` and every `docs/decisions/D-*.md`
currently `proposed` — explicitly tell each agent "you are running in
design mode" (per their own frontmatter, this is not inferred). Write each
raw reply verbatim to `work/design/artifacts/<agent>-verdict-raw.json`
(handoff re-entry mode: the suffixed path named there instead), then:

```
${CLAUDE_SKILL_DIR}/../../scripts/verdict-filter \
  work/design/artifacts/<agent>-verdict-raw.json \
  --out work/design/artifacts/<agent>-verdict.json --project <project>
```

Write `work/design/design-review.md` (same spirit as `verify.md` — quotes
the filtered files, doesn't paraphrase them): each adversary's `attacked`
list, each kept verdict, the kept/dropped counts. **Count dropped verdicts
explicitly here** — a verdict `verdict-filter` dropped for citing a
decision id that doesn't resolve, or a quote that doesn't match, is itself
worth one line (it means an adversary's evidence discipline slipped, worth
noticing even though the verdict itself never reaches the human).

## 6. Resolve findings — the human's call, every time

For each kept verdict: present it to the human — but the verdict's own
claim/evidence text alone is not enough to present. A human triaging a
stream of these is not re-reading the whole charter and design-review.md
in parallel to reconstruct why a given finding matters; that
reconstruction is your job, not theirs. Every finding you present must
include, inline, in addition to the verdict itself:

- **Which charter guarantee is actually at stake** — the specific
  non-negotiable/hard-constraint line this finding threatens, quoted or
  closely paraphrased, not just "see docs/charter.md."
- **What breaks in practice if this stays as-is** — one concrete sentence:
  a specific user action or scenario and its bad outcome, not a repeat of
  the verdict's own abstract claim.

Group related kept verdicts (e.g. a paired citation from the same
scenario, or two adversaries independently flagging the same gap from
different angles) into one presented finding rather than asking about
each verdict line in isolation — the human is resolving *findings*, and a
finding that took four verdict entries to state fully is still one
decision for them to make.

Two outcomes, both real, neither silent:

- **Revise** — amend the cited decision (supersede it — append-only, per
  `docs/decisions/decision.md`'s own lifecycle rules; never edit an
  existing record's content in place) or add/adjust a `DEFERRED.md` entry
  if the finding reveals the category wasn't actually decidable yet.
- **Recorded override** — the human disagrees with the verdict and wants
  to proceed as designed anyway. Do not force a revision. Append a
  `## Design review overrides` section to `work/design/design-review.md`
  (or this pass's own suffixed path, in handoff re-entry mode — see
  "Handoff re-entry mode" above) quoting the verdict and the human's
  stated reason — this is the same
  trust model `/ship --bypass` already uses (loud, recorded, never
  silent), applied one tier earlier.

Only after every kept verdict has one of these two outcomes: flip every
`docs/decisions/D-*.md` still at `proposed` (that wasn't superseded during
revision) to `- Status: adopted`. This is the one and only place `/design`
flips that status — a `proposed` decision that never went through design
review must never become `adopted` by any other path.

## 7. Stopping rule

**(Handoff re-entry mode uses its own completion check instead — see
"Handoff re-entry mode" above. Don't run `design-gate` in that mode.)**

```
${CLAUDE_SKILL_DIR}/../../scripts/design-gate --project <project>
```

If it fails: fix the specific thing it names (an uncovered category, an
unplanned capability, milestone 0's capability-targets table, or too many
adopted decisions — the last one means splitting a decision that's really
several, or genuinely deferring some of it) and re-run. **Do not hand off
while this fails** — this is the mechanical version of "the skeleton-skip
anti-pattern must be impossible, not discouraged."

## 7.5. Write the design summary

Write `docs/design-summary.md` from
`${CLAUDE_SKILL_DIR}/../../templates/design-summary.md` — a plain-prose
walkthrough of the charter's shape plus every adopted decision, written
for a human engineer skimming once before their first milestone task, not
for grep or citation resolution. This is genuinely a different document
from `docs/decisions/D-*.md`: those are precise and machine-consumable
(decision-hash, verdict-filter citations); this one exists because a
human doesn't read eight of those start-to-end to get the gestalt of what
was decided. Don't paraphrase a decision's full record into this file —
one short paragraph per adopted decision (what it means practically, not
its Context/Alternatives-rejected text) ending in a `(see D-<n>)` pointer
back to the real record. Skip a foundational category entirely here if it
has no adopted decision, only a `DEFERRED.md` entry — list those under
this file's own "Open questions" section instead of padding the main
walkthrough. Regenerate this file (never hand-patch it) if a later
decision supersedes one it summarizes.

## 7.6. Regenerate the decision index

```
${CLAUDE_SKILL_DIR}/../../scripts/decision-index --project <project>
```

Every `D-*.md` this stage wrote or flipped to `adopted` in §6 must be
reflected in `docs/decisions/INDEX.md` before it's committed alongside
them — never let the index ship a stage behind the store it's supposed to
summarize. Mechanical, no review needed; see the script's own header for
why this is a triage aid, never a citation target.

## 8. Commit and hand off

**(Handoff re-entry mode uses its own commit and hand-off text instead —
see "Handoff re-entry mode" above. What follows is for a first run,
`--handoff`-grounded or not.)**

One commit — same untrailered, setup-shaped precedent `/bootstrap`'s own
install commit already uses (this is design-stage setup, not a task; there
is no `Spine-Task:` id to attach yet):

```
git add -A -- docs/charter.md docs/design-summary.md docs/decisions/ \
  work/M0/ work/design/ .spine/capabilities.json .spine/adapters/
git commit -m "spine: design stage — <n> decisions adopted, milestone 0 defined"
```

Tell the human: how many decisions were adopted (and the cap they're
against, from `design-gate`'s own output), what's in `DEFERRED.md` and
each item's trigger, point at `docs/design-summary.md` as the one-page
read before diving into decision records, and that
`/task <description> --milestone M0` is the next real command — the first
member task of the walking skeleton. This is a non-recurring event (new
touchpoints must not become recurring ones) — `/design` runs once per
project (or once per brownfield
adoption pass, not exercised by this build), never per task.
