---
name: design
description: The design stage — a facilitated, interactive session that turns a confirmed charter into foundational decisions, a walking-skeleton milestone, and a reviewed stopping point before any feature code exists. Run after /bootstrap on greenfield; may also run on brownfield to make implicit architecture explicit (not exercised by this build).
disable-model-invocation: true
argument-hint: [--project <path>]
---

You are running `/design` against `--project <path>` from `$ARGUMENTS`
(default: current directory). Scripts at
`${CLAUDE_SKILL_DIR}/../../scripts/<name>`, templates at
`${CLAUDE_SKILL_DIR}/../../templates/<name>`, agents `falsifier`/`security`
in design mode (`core/agents/falsifier.md`/`security.md` §"Design-mode
mandate").

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

## 1. Walk the six foundational categories

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

## 5. Design review

Run both adversaries in design mode, fresh `Agent` calls
(`subagent_type: falsifier` / `subagent_type: security`), delegation
message pointing at `docs/charter.md` and every `docs/decisions/D-*.md`
currently `proposed` — explicitly tell each agent "you are running in
design mode" (per their own frontmatter, this is not inferred). Write each
raw reply verbatim to `work/design/artifacts/<agent>-verdict-raw.json`,
then:

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

For each kept verdict: present it to the human. Two outcomes, both real,
neither silent:

- **Revise** — amend the cited decision (supersede it — append-only, per
  `docs/decisions/decision.md`'s own lifecycle rules; never edit an
  existing record's content in place) or add/adjust a `DEFERRED.md` entry
  if the finding reveals the category wasn't actually decidable yet.
- **Recorded override** — the human disagrees with the verdict and wants
  to proceed as designed anyway. Do not force a revision. Append a
  `## Design review overrides` section to `work/design/design-review.md`
  quoting the verdict and the human's stated reason — this is the same
  trust model `/ship --bypass` already uses (loud, recorded, never
  silent), applied one tier earlier.

Only after every kept verdict has one of these two outcomes: flip every
`docs/decisions/D-*.md` still at `proposed` (that wasn't superseded during
revision) to `- Status: adopted`. This is the one and only place `/design`
flips that status — a `proposed` decision that never went through design
review must never become `adopted` by any other path.

## 7. Stopping rule

```
${CLAUDE_SKILL_DIR}/../../scripts/design-gate --project <project>
```

If it fails: fix the specific thing it names (an uncovered category, an
unplanned capability, milestone 0's capability-targets table, or too many
adopted decisions — the last one means splitting a decision that's really
several, or genuinely deferring some of it) and re-run. **Do not hand off
while this fails** — this is the mechanical version of the build prompt's
own "the skeleton-skip anti-pattern must be impossible, not discouraged."

## 8. Commit and hand off

One commit — same untrailered, setup-shaped precedent `/bootstrap`'s own
install commit already uses (this is design-stage setup, not a task; there
is no `Spine-Task:` id to attach yet):

```
git add -A -- docs/charter.md docs/decisions/ work/M0/ work/design/ \
  .spine/capabilities.json .spine/adapters/
git commit -m "spine: design stage — <n> decisions adopted, milestone 0 defined"
```

Tell the human: how many decisions were adopted (and the cap they're
against, from `design-gate`'s own output), what's in `DEFERRED.md` and
each item's trigger, and that `/task <description> --milestone M0` is the
next real command — the first member task of the walking skeleton. This
is a non-recurring event (per the build prompt's constraint that new
touchpoints must not become recurring ones) — `/design` runs once per
project (or once per brownfield adoption pass, not exercised by this
build), never per task.
