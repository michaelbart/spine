# Extension Build — Phase E Handoff

Date: 2026-08-10. Builder: Claude Code (Sonnet 5), interactive session
(fresh session from Phase D, per the standing session-boundary discipline).

Read order for anyone resuming or auditing: `ext-phase-A-handoff.md`,
`ext-phase-B-handoff.md`, `ext-phase-C-handoff.md`, `ext-phase-D-handoff.md`,
this file, then `docs/tradeoffs.md` itself (the fuller written record —
this handoff points at what changed and why, not a duplicate of the
content).

**This is the final phase. Per the build prompt's own §9, Phase E's scope
is documentation only — tradeoffs extensions, self-red-team, v2 shelf with
insertion points — and it is now delivered.**

## 1. What was built

Two files, both in `spine/`, nothing else touched:

| Item | Status |
|---|---|
| `work/.build/build-prompt-extensions.md` | New — the literal text of the extension build prompt, sourced directly from the engineer this phase (it was not saved anywhere on disk through Phases A–D; confirmed by search before asking — see §2 of `ext-phase-D-handoff.md`, which first named this gap and recommended closing it). Saved verbatim, unedited. |
| `docs/tradeoffs.md` | Extended — 513 lines added, zero lines removed or altered outside the insertion points. `git diff --stat`: `1 file changed, 513 insertions(+)`. |

`git status --short` in `~/spine` before writing this handoff shows exactly
those two files (plus an unrelated, pre-existing `.DS_Store`, not part of
this build). No core script, hook, skill, template, or agent file was
touched this phase — matching the build prompt's own scope for Phase E.

### `docs/tradeoffs.md` additions, by section

- **`## Extension Build — the design stage and multi-repo coordination`**
  (new top-level section, inserted after the original build's worked
  example and before `## Self-red-team`): design-stage cost estimate,
  multi-repo cost estimate, the maximal-ceremony hazard (disclosed as
  **not fully exercised** by this build's own two worked examples — see
  §3 below, this is the phase's most important honesty call), four
  residual risks (registry-neglect, the inconsistency window, the
  affected-repo residual, the decision-store residual), a full write-up of
  the greenfield worked example (`~/bookmarks` through `/design` and its
  skeleton), a full write-up of the multi-repo worked example
  (`bookmarks`/`bookmarks-cli`/`bookmarks-workspace`, both the additive
  ship and the breaking-change refusal), the single-repo regression
  demonstration (including the new project-trust primitive finding from
  Phase D §6), and all seven extension open questions (build prompt §5)
  answered and defended in the same style as the original seven.
- **`## v2 shelf`** — 11 new rows: `contract-scan`, baseline failure
  attribution, decision-drift metrics in `/costs`, per-repo charters,
  auto-generated topology maps, brownfield `/design` (unexercised, not
  unbuilt), multi-workspace, the `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_
  CLAUDE_MD` env-flag dependency (explicitly out of scope per build prompt
  §0.3), full `/costs` workspace aggregation (open question 6), design-mode
  adversary cost tiers (open question 7).
- **`## The Auto Mode classifier wall`** — one new paragraph, appended
  after the existing Phase E (original build) confirmation: the nested-
  `claude`-invocation finding from `ext-phase-A-handoff.md` §3, restating
  the wall's scope as "headless sessions, or nested `claude` invocation
  from within any session" rather than "headless sessions" alone.
- **`## Self-red-team`** — one new subsection, `### Extension A + B —
  self-red-team`, covering every item the build prompt §8 asks for by
  name (breaking-change misclassification — real evidence, tested;
  grounding on a quarantined decision — disclosed as unenforced, same
  shape as the existing charter risk; a `contract-check` adapter validating
  too little while passing conformance — real evidence, the `title_length`
  Unicode bug; the design cap gamed by cramming — disclosed as untested;
  registry staleness gamed a different way than what was tested — disclosed;
  hook-firing watched in the workspace topology — confirmed; capabilities
  `implemented` without conformance, including the new #14 — confirmed
  none; the regression check — cross-referenced), plus a "real bugs found
  and fixed" list pulling together every concrete defect Phases B–D's own
  testing surfaced in spine core (the two bash-3.2 empty-array bugs, the
  `conformance` untracked-files gap, the `contract-touch` classifier's
  markdown-bullet false negative, the `floor`/`secret-scan` dispatch bug,
  the missing repo-qualification on `grounding-decisions:`, the
  `contract-check`-vs-floor-eligibility conflation, and the two
  bookmarks-local adapter bugs the falsifier caught).
- **`### Single-repo regression demonstration`** (inside the new Extension
  Build section) also resolves a forward reference the same edit
  introduced: the `~/bgr` `secret-scan`-on-zero-diff failure (named but not
  detailed in `ext-phase-D-handoff.md` §6) is now written up in full,
  cross-referenced against "Two-stack validation" (the original build's
  section documenting the same underlying full-tree-fallback shape) —
  explicitly attributed to the *original* build's scope, not Extension B,
  since `floor` itself is unmodified by Extension B.

## 2. What was verified

This phase wrote documentation only; there was nothing to fire-test. What
was checked instead:

- **Every real event described in the new tradeoffs.md content is drawn
  directly from `ext-phase-A/B/C/D-handoff.md`**, not invented or
  extrapolated — each claim (verdict counts, bugs found, commands run,
  files touched) traces to a specific passage in one of those four
  handoffs, all of which this phase read in full before writing anything.
- **The build prompt saved this phase matches the four handoffs' own
  citations of it exactly** — cross-checked before saving: the §0
  primitive-test descriptions match `ext-phase-A-handoff.md` §1 verbatim
  in substance; the §2 architecture matches what Phases B/C/D actually
  built; the §5 open-questions numbering (1–7) matches every "open question
  §5.n" reference across all four handoffs; the §9 phase structure matches
  the five phases actually run. No contradiction found between the saved
  prompt and the four handoffs' accounts of it.
- **`git status --short` / `git diff --stat`** confirm the scope of this
  phase's changes precisely matches what's described in §1 above — no
  accidental edits to core files.

## 3. Decisions made and reasoning

### 3.1 The build prompt genuinely wasn't on disk, and had to be re-sourced from the engineer a second time

`ext-phase-D-handoff.md` §5 already named this as a real, load-bearing
finding — the build prompt's literal text was absent from `spine/`,
`~/Downloads`, `~/Documents`, `~/Desktop`, and every reasonable location a
search covered, both when Phase D first hit this gap and again when this
phase re-checked before asking the engineer. (One unrelated file was
found during this phase's own search —
`~/Downloads/keeperfiles/build-prompt-phase-0-1.md` — read and confirmed
to belong to a different, unrelated project before being disregarded.)
**Decision, matching Phase D's own recommendation**: ask the engineer
directly rather than attempt to reconstruct the prompt's exact wording
from the four handoffs' paraphrases, since verbatim §0–§9 text (exact
architecture wording, the exact anti-pattern list, the exact self-red-team
requirements) is precisely what a paraphrase-only reconstruction would
lose, and losing it is exactly the failure mode Phase D's own finding
warned about. The engineer supplied the full text; it was verified against
all four handoffs for consistency (§2 above) before being saved.

### 3.2 The maximal-ceremony hazard is disclosed as untested by this build's own worked examples, not asserted as mitigated

This is the single most consequential editorial decision this phase made.
The build prompt names a specific risk — design-stage ceremony, workspace-
init ceremony, and skeleton-milestone ceremony compounding before anything
runs, on a *new* multi-repo greenfield project's first day — and asks the
tradeoffs doc to cover "the sequencing that mitigates it" (build prompt
§6). Reading the four handoffs closely shows the two worked examples never
actually composed that way: `/design` and the skeleton shipped against
`~/bookmarks` as a single repo in Phase C, before any workspace existed;
the workspace was built in Phase D around that already-skeleton-shipped
repo plus a fresh `bookmarks-cli` that was bootstrapped directly, never run
through `/design` itself. The honest thing to write is not "the mitigation
holds" — no worked example proves that — but that the mitigation is
architecturally present and the specific composed scenario the build
prompt is worried about has zero real-worked-example evidence behind it
either way. Recorded exactly that way in the new tradeoffs.md section,
rather than smoothing over the gap between what the build prompt asked to
be demonstrated and what the actual phase history demonstrates.

### 3.3 Wrote up genuinely untested branches as untested, not as covered

Two places in the new self-red-team subsection say plainly that something
was not exercised: the design cap gamed by cramming multiple decisions
into one record (build prompt §8 asks this be named explicitly; no
evidence either way exists in any handoff), and the affected-but-not-edited
floor-eligibility branch of `contract-check` (the one real multi-repo task
this build ran had its consumer repo genuinely edited, not merely
affected, so the "gated on `contract-check` alone, full floor skipped"
branch never actually ran against a repo carrying pre-existing baseline
failures). Both are recorded as real, disclosed gaps in this build's own
evidence — consistent with the standing instruction throughout this build
(and restated in this build prompt's own closing line) to write only what
would be defended to a skeptical principal engineer.

### 3.4 Placement: one new top-level section, not scattered edits throughout

Considered interleaving Extension A/B content throughout the existing
single-repo sections (e.g., adding decision-lifecycle material into "What
this costs," contract material into "Residual risks"). Decided against it:
the original document is a complete, coherent account of a finished
single-repo build, and interrupting it with extension-specific caveats
scattered throughout would make it harder to read as either document. One
new top-level section, placed after the original worked example and before
the (now-extended) self-red-team synthesis, keeps both builds' accounts
intact and lets the self-red-team section do the work of tying old and new
findings together at the end, which is where a reader actually wants the
synthesis.

## 4. What a future session needs that isn't obvious from the files

- **This build is now fully delivered** — five phases (A–E) of the
  extension build prompt, on top of the six phases (A–E, where the original
  build's own Phase E was itself already a post-delivery fix pass) of the
  original build. `spine/`'s own git history has one commit per phase from
  `Extension A, Phase B` (`5c4d9ad`) through `Extension B, Phase D`
  (`d1736e3`) — **this phase's own changes (`docs/tradeoffs.md`, the saved
  build prompt) are not yet committed.** Whether and how to commit them is
  the engineer's call, not decided unilaterally here, matching this
  build's own standing practice around commit-ordering decisions (see
  `phase-D-handoff.md` §3.2 for the precedent).
- **`~/bookmarks`, `~/bookmarks-cli`, and `~/bookmarks-workspace` remain
  real, live, spine-installed projects** with real git history — nothing
  in this phase touched any of them. `work/M1/milestone.md` in
  `~/bookmarks-workspace` still has two real, disclosed `TBD` legs
  (migrate, contract) from Phase D, unchanged.
- **No new primitive findings, bugs, or worked examples exist beyond what
  Phases A–D already produced** — this phase's job was synthesis and
  disclosure of what already happened, not new building or testing. Anyone
  auditing this build's factual claims should trace them to the four prior
  handoffs, not to this one.
- **The two "disclosed as untested" items in §3.3 above are the most useful
  starting points for whoever next extends this system** — the composed
  maximal-ceremony scenario (a genuine new multi-repo greenfield project
  run through `/design` → `/workspace` → skeleton milestone in one
  continuous flow) and the affected-but-not-edited `contract-check` branch
  are both real, well-scoped, and would directly close open evidentiary
  gaps rather than open new architectural questions.
