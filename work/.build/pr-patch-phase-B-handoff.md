# PR-description patch — Phase B: template, wiring, demonstration

Builds on `pr-patch-phase-A-handoff.md` (approved, three dispositions
folded in below). Delivers: `core/templates/pr-description.md` (new),
`core/scripts/conformance` (bookkeeping-exclusion fix, disposition 1),
`core/skills/ship/SKILL.md` (new `## 4a`), `README.md` (one clause),
`docs/tradeoffs.md` (new self-red-team section) — and, executed, not
narrated: a real bug-fix regression check, three demonstration PR
descriptions (one real-and-shipped, one real-but-never-shipped, one
labeled synthetic), a full real task run through the entire `/ship` loop
including its new step, and a sentence-by-sentence trace audit.

## What shipped

| File | Change |
|---|---|
| `core/scripts/conformance` | Excludes `work/**` and `.spine/current-task` from `actual` before scoring (disposition 1); states the exclusion rule in stdout and in the `--out` JSON's new `excluded`/`excluded_rules` fields |
| `core/templates/pr-description.md` | New. The four-unions "Where to look" rule, the completeness line, the empty-case line, all in the template's own header comment so they travel with the artifact |
| `core/skills/ship/SKILL.md` | New `## 4a. Write the PR description`, between `## 4` and `## 5` (no renumbering — see Phase A §1); one line added to `## 6` pointing at `pr-description.md` alongside the existing briefing pointer |
| `README.md` | `/ship`'s command-reference row: one added clause |
| `docs/tradeoffs.md` | New "PR-description patch — self-red-team" section; one amendment joining the readability patch's existing multi-repo watch item |

## Disposition 1 — conformance fixed at the source, regression-checked for real

`core/scripts/conformance` now excludes `work/**` and `.spine/current-task`
from its `actual` set before computing precision/recall/f1, and states the
exclusion rule in both outputs (`... (excluded N bookkeeping path(s) from
actual: work/**, .spine/current-task)` on stdout; `excluded`/
`excluded_rules` fields in the JSON). Union (4) in the PR-description
template now reads clean data with zero display-layer filtering of its
own — confirmed by re-reading the fixed script's own header comment against
the template's union (4) text, they agree on where the filtering happens
(once, at the source).

**Real regression check, against the real, already-shipped
`20260811-extract-slugify-helper` record (`~/bgr`):**

```
$ /Users/michaelbart/spine/core/scripts/conformance \
    ~/bgr/work/20260811-extract-slugify-helper/plan.md \
    --project ~/bgr --base 72927a8^ --out /tmp/conformance-slugify-postfix.json
conformance: predicted=2 actual=2 precision=1.00 recall=1.00 f1=1.00 \
  (excluded 15 bookkeeping path(s) from actual: work/**, .spine/current-task)
```

Before the fix (the real, original `conformance.json` this task shipped
with): `predicted=2 actual=14 precision=1.00 recall=0.14 f1=0.25` — 12 of
the 14 "actual" files were the task's own `work/<task-id>/**` bookkeeping.
After: `predicted=2 actual=2 precision=1.00 recall=1.00 f1=1.00` — the real
signal (both predicted files, and only those, were touched) that was there
all along. The real task's `artifacts/conformance.json`, `verify.md`'s
Conformance line, and `ledger.json`'s `conformance_score` were all updated
to the post-fix values (not just the ad hoc `/tmp` run) — this is a real
task's real record, corrected once its metric was fixed, not a synthetic
demo.

**Explicitly not comparable, recorded in `docs/tradeoffs.md`:** any
`/costs` trend spanning this fix's ship date mixes structurally-deflated
pre-fix scores with real post-fix ones — a discontinuity to treat as such,
never as a genuine quality jump.

## Disposition 2 — deviation-extraction heuristic, ratchet armed, not built

Kept as v1 heuristic in the template (backtick-wrapped, path-shaped token
extraction from `deviations.md` prose). Not implemented: a structured
`- Files:` field on `core/templates/deviations.md`. Armed instead: the
first real miss (not the near-duplicate found and fixed below — an actual
case where the heuristic fails to surface a file a deviation genuinely
cites) is instance one; two real misses is `/ratchet`-eligible, and the
response is already decided in `docs/tradeoffs.md`. Recorded there, not
just here, so it survives past this handoff.

**One real, non-miss finding during demonstration, fixed at the heuristic
level (not deferred to the ratchet):** the real slugify deviation record
produces both `` `slug.test.ts` `` and `` `src/lib/slug.test.ts` `` as
separate backtick-wrapped matches — the same file, cited twice at
different qualification levels in the same record. Added a same-record
suffix-collapse tiebreaker (keep the longer, more path-qualified token) to
the template's own rule 1. This is a duplicate/noise fix, not a miss, so
it didn't need the ratchet — but it's exactly the kind of real edge a
"heuristic, not a parser" produces, which is why the trigger stays armed
for the next one that isn't this shape.

## Disposition 3 — the completeness line, proven against a real high-severity finding

`core/templates/pr-description.md`'s union (2) rule: a kept adversary
verdict with `evidence.kind == "command"` contributes nothing to "Where to
look" — no file to point at — and the list's own last line becomes `"plus
N finding(s) without file anchors — see verify.md"` whenever `N > 0`.

**Proven against the real horizon record**
(`~/horizon/work/20260808-fix-building-group-delete-orphans-units/`,
`artifacts/security-verdict-raw.json`): its own single highest-severity
finding — a `.claude/settings.json` permission-escalation bleed-in,
`severity: high`, `evidence.kind: command` — does **not** get a
where-to-look entry under this rule. See the generated description below:
it appears in full under "How it was verified → Adversaries" (never lost)
and the where-to-look list's last line correctly reads `"plus 3 finding(s)
without file anchors — see verify.md"` (2 falsifier command-evidence
verdicts + this 1 security one). This is the sharpest real test available
in this environment for "the list must never imply a completeness it
doesn't have" — a naive implementation that dropped `command`-evidence
findings silently would have buried exactly the finding most worth seeing,
and the demonstration confirms the completeness line catches it instead.

**A second, related guard found during the same demonstration, not
anticipated in Phase A:** re-running the fixed `conformance` against
horizon's real (old-template) `plan.md` produces `precision=0.00` —
its `## Predicted touch` entries are Markdown-backtick-wrapped (the
separate, already-documented pre-existing bug the readability patch's
template fix guards against going forward, not retroactively). A raw
union-(4) set difference against that corrupted `predicted` list would
have falsely flagged all 3 genuinely-predicted files as drift. Guarded in
the template: any backtick inside a `predicted` entry makes union (4)
declare itself unavailable for that task ("plan accuracy not computable —
predicted-touch entries are Markdown-formatted") rather than emit the
wrong answer. See the generated horizon description below — this is the
reason its "Plan accuracy" bullet reads "not computable," not "0.00."

## Demonstration 1 — real, shipped task: `20260811-extract-slugify-helper`

Full generated file: `~/bgr/work/20260811-extract-slugify-helper/pr-description.md`
(written into the real task's own folder — exactly where `/ship` §4a would
place it). `ledger set 20260811-extract-slugify-helper pr_description
"generated"` run for real against the real ledger.

Union computation, shown against the real artifacts:
- **(1) Deviations**: `deviations.md` has one real record. Extraction
  pulled `` `src/lib/slug.test.ts` `` (from "Resolution") and
  `` `work/20260811-extract-slugify-helper/plan.md` `` (from "Plan
  assumed") — both real backtick-wrapped path tokens, confirmed by
  re-reading the record. The suffix-collapse tiebreaker fired once (see
  disposition 2).
- **(2) Adversaries**: none ran for this task (verify.md's own
  disclosure) — zero entries, `N = 0`.
- **(3) Protected paths**: real diff is `src/lib/slug.ts`,
  `src/lib/slug.test.ts` — checked against `~/bgr/.spine/protected-
  paths.conf`'s real globs (`prisma/**`, `src/lib/auth*.ts`,
  `src/components/auth/**`, `middleware.ts`, admin paths,
  `src/lib/aggregation.ts`, manifests) — no match. Zero entries.
- **(4) Conformance drift**: post-fix `actual` = `predicted` exactly
  (disposition 1's regression check). Zero entries.

Combined: 2 entries (both from union 1), not the empty-case line — a real,
non-trivial exercise of the mechanism.

## Demonstration 2 — real record, never shipped: `20260808-fix-building-group-delete-orphans-units`

Full generated file:
`spine/work/.build/pr-description-demo/horizon-pr-description.md` (written
into spine's own demo directory, not horizon's real task folder — this
task's real `/ship` run never reached §4a, the merge gate correctly
blocked it at the floor, and writing a "shipped" artifact into another
project's real task folder would misrepresent that task's real state; same
disclosure precedent the readability patch used re-rendering this task's
plan/briefing). The file's own header states this plainly, and the
rendered body repeats it in its own "Status" line.

Union computation:
- **(1) Deviations**: no `deviations.md` exists for this task (predates
  the discipline). Zero entries — disclosed in the rendered "What
  surprised us," not silently absent.
- **(2) Adversaries**: 7 kept verdicts total across both agents (no
  `verdict-filter` run — human fallback reported all raw output). 4 carry
  `file_line` evidence → 3 union entries after deduping the TOCTOU race
  both adversaries independently found at the same file:line (merged into
  one entry citing both). 3 carry `command` evidence → excluded, `N = 3`,
  the completeness line fires (disposition 3).
- **(3) Protected paths**: real diff (`.claude/settings.json`,
  `.spine/adapters/{callers,lint,typecheck}`, the 3 `lib/` files) checked
  against `~/horizon/.spine/protected-paths.conf`'s real globs
  (`firestore.rules`, `lib/features/authentication/**`, `admin/**`,
  manifests, etc.) — no match. Zero entries.
- **(4) Conformance drift**: unavailable — backtick-corrupted `predicted`
  guard fired (disposition 3's second finding).

Combined: 3 file entries + the completeness line — the richest real
exercise of the mechanism available in this environment.

## Demonstration 3 — labeled synthetic empty-case

Full generated file:
`spine/work/.build/pr-description-demo/empty-case/pr-description.md`, with
its own supporting `plan.md`/`deviations.md`/`verify.md`/
`artifacts/{conformance,falsifier-verdict}.json` in the same directory —
every one labeled `SYNTHETIC` in its own header, engineered so all four
unions compute to empty (zero deviations, one adversary ran and kept zero
verdicts, no protected-path touch, `actual == predicted` exactly). Renders
the template's fixed line: *"The diff landed exactly where the approved
plan predicted; no deviations, findings, or drift. Spot-check at will."*

**A second, real (non-synthetic) empty-case instance turned up during
Demonstration 4 below** (the `20260811-add-truncate-helper` regression
task) — real confirmation the line isn't only reachable by construction.

## Demonstration 4 — regression: a real trivial task through the full `/ship` loop, including §4a

A new, real, small task in `~/bgr`: `20260811-add-truncate-helper` — a
pure `truncateWithEllipsis(str, maxLen)` helper, Class 1 (the minimum
class that reaches `/ship` at all — Class 0 skips the whole apparatus, so
proving §4a fires needed at least Class 1), run by hand through every real
phase:

- Classified, task folder created for real (`research.md`, `plan.md` on
  the current template, `claims.json`, `owner`, `approval.json`,
  self-approved — expected and correct for Class 1 per `task/SKILL.md`
  §3).
- Implemented for real: `src/lib/truncate.ts` + `src/lib/truncate.test.ts`
  (4 cases: under-length, exact-length boundary, over-length, empty).
- `npx vitest run src/lib/truncate.test.ts` → **4 passed (4)**, real
  output.
- `core/scripts/floor 1 --task 20260811-add-truncate-helper --project
  ~/bgr` → **PASS (7/7)**. One honest note carried into `verify.md`:
  `typecheck`/`lint` scored vacuous-pass because both new files are
  untracked and those two adapters scope to `git diff` against *tracked*
  files — a pre-existing `~/bgr`-adapter behavior, unrelated to this
  patch, disclosed rather than silently accepted as a clean pass.
- `core/scripts/conformance` (the fixed version) → real
  `predicted=2 actual=2 precision=1.00 recall=1.00 f1=1.00`.
- Merge gate (§1): floor PASS confirmed in `verify.md`, `grep -c '^-
  Status: open' deviations.md` → real `0`. Cleared.
- §2 (decisions): plan has no `## Grounds on decisions`, no resolved
  deviation to distill — correctly skipped, nothing written.
- §3 (milestone): no `work/<task-id>/milestone` file — correctly skipped.
- §4: real `briefing.md` written.
- **§4a (this patch's own new step): real `pr-description.md` written.**
  All four unions computed to empty for real (no deviations, no
  adversaries run, no protected-path touch, clean conformance) — renders
  the fixed empty-case line, the second, real confirmation of
  Demonstration 3's synthetic proof.
- `ledger init` + `ledger set ... pr_description "generated"` — real
  ledger calls, real output shown, real `ledger.json` on disk.
- §5 staging: `git add` the task's own real changed paths — confirmed via
  `git status --short` that exactly this task's files (plus the
  disposition-1 regression's edits to the already-shipped slugify task)
  were staged, nothing else.

**Stopped short of the real `git commit`, deliberately** — same call the
readability patch made shipping its own slugify demo ("only commit when
explicitly asked"; nobody asked for a commit in `~/bgr` this session).
Unstaged everything back out (`git reset`) rather than leave a
half-finished stage. `~/bgr`'s working tree now has this real, uncommitted
change plus this session's disposition-1 regression edits to the
already-shipped slugify task's artifacts; `work/20260811-add-
truncate-helper/state` reads `ship` (not `done` — `/ship` §6 only writes
`done` after every commit in the sequence actually lands, which didn't
happen here), and `.spine/current-task` still points at this task. Your
call whether to commit it for real or `git checkout`/clean it back out.

## Trace audit — every sentence in Demonstration 1's generated description

| Line(s) | Claim | Source |
|---|---|---|
| Title | Task ID + title | `plan.md`'s own `# Plan:` header, verbatim |
| What & why, sentence 1 | `slugify()` extracted, was untestable inline | `plan.md` `## The gist` |
| What & why, sentence 2 | Both call sites unchanged | `plan.md` `## The gist`, closing clause |
| Review this at plan level, sentence 1 | Pure extraction mechanics | `plan.md` `## The gist` |
| Review this at plan level, sentence 2 | The one named risk (regex precedence) | `plan.md` `## What could go wrong` |
| Review this at plan level, sentence 3 | No DB/schema/dependency touch | `plan.md` `## The gist` + `## What could go wrong` |
| Floor bullet | pass, 7/7 capabilities | `verify.md` Floor results table |
| Adversaries bullet | not run, deliberate scope limit | `verify.md` `## Adversary verdicts` |
| Plan accuracy bullet | precision/recall/f1 1.00/1.00/1.00 | `work/.../artifacts/conformance.json` (post-fix) |
| Gaps bullet | mutate/smoke-*/migrate-rehearse unavailable | `verify.md` `## Capability gaps` |
| Where to look, entry 1 | `slug.test.ts` — punctuation fallback confirmed | `deviations.md` Deviation 1, "Resolution" (union rule 1) |
| Where to look, entry 2 | `plan.md` — the anticipating tier | `deviations.md` Deviation 1, "Plan assumed" (union rule 1) |
| What surprised us | the record-and-proceed deviation, condensed | `deviations.md` Deviation 1, all fields |
| Record line | pointer to the task folder | Fixed template closing line |

**Zero sentences without a traced source.** No claim in this description
was generated by reading `src/lib/slug.ts` itself — confirmed by this
audit never citing the source file as a claim's origin, only as a target
the artifacts point at.

## Format compromises / honest gaps

- **Multi-repo (`**Contracts**` section, the one-shared-description
  decision)**: shipped structurally complete, undemonstrated against real
  multi-repo data — no `workspace.json` exists anywhere in this
  environment (confirmed again during this phase). Joins the readability
  patch's existing watch item in `docs/tradeoffs.md`, not a separate one.
- **`typecheck`/`lint` vacuous-pass on new untracked files** (found during
  Demonstration 4): a pre-existing `~/bgr`-adapter scoping behavior, real
  and disclosed in that task's own `verify.md`, out of scope for this
  patch (it's a project adapter implementation detail, not
  `core/scripts/conformance` or the PR-description assembly logic).
- **`~/bgr` and `~/horizon` working trees carry real, deliberately
  uncommitted demonstration state** — listed precisely in Demonstration 4
  above. Nothing pushed, nothing force-anything, all easily inspected or
  reverted.

## Self-red-team

Full write-up in `docs/tradeoffs.md`'s new "PR-description patch —
self-red-team" section (the reviewer-rubber-stamp risk, where-to-look
editorial drift as a ratchet candidate, the deviation-heuristic armed
trigger, the completeness-line proof, the stale-description v1 behavior,
and the conformance-score discontinuity note) — not duplicated here per
this patch's own no-re-derivation rule: one artifact states it, this
handoff points at it.

## Pin-bump note for the maintainer

Two real script/template changes this time, not template-prose-only like
the readability patch: `core/scripts/conformance`'s bookkeeping-exclusion
fix changes real output for every consumer (every future `/ship` briefing's
"Plan accuracy" bullet, `/costs`' `conformance_score` aggregate) — read
the discontinuity note above before trusting a `/costs` trend that spans
this bump. `core/templates/pr-description.md` is new; `ship/SKILL.md`
gained `## 4a` without renumbering anything (verified: `§0`/`§1`/`§4`/`§5`/
`§6` cross-references in `claims-check`, `writing-mandate.md`,
`docs/tradeoffs.md`, and the two-engineer demo's `deviations.md` all still
resolve correctly — none of them pointed at `§4a`, and none needed to).
No hooks, agents, or capabilities changed; `adapter-conformance --all` is
unaffected. Before bumping `.spine/core-pin.json` in `~/bgr` or
`~/horizon`: re-run this patch's own regeneration on that project's most
recent real task if one exists (cheap — the same `conformance`/`ledger`
commands demonstrated above), same recommendation the readability patch
made and for the same reason.

---

**Stop. Deliver.**
