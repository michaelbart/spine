# Extension Build — Phase B Handoff

Date: 2026-08-08. Builder: Claude Code (Sonnet 5), interactive session
(fresh session from Phase A, per the standing session-boundary discipline).

Read order for anyone resuming: `ext-phase-A-handoff.md`, this file, then
the files themselves (all listed below, all real, all demonstrated firing
in this session).

Scope: build prompt §9 Phase B — "Extension A's deterministic layer:
check-stale branch, verdict-filter `decision:` type, design-gate,
templates, ship-side decision lifecycle — each demonstrated with a real
invocation." Nothing from Phase C (`/design`, milestone-loading in
`/task`, adversary design-mode mandates) or Phase D (workspace/Extension B)
was touched.

## 1. What was built

### New files

- **`core/scripts/decision-hash`** — `sha256(file content, minus the
  `- Status:` line)`. Single source of truth for the algorithm; both
  `check-stale` and (from Phase C onward) any skill computing a citation
  hash must shell out to this rather than reimplementing it. Portable:
  tries `sha256sum`, falls back to `shasum -a 256` (macOS has no
  `sha256sum` by default).
- **`core/scripts/design-gate`** — the four stopping-rule checks from the
  build prompt §2's "Stopping rule," each independently mechanical:
  1. `work/M0/milestone.md` exists, its Capability targets table has a
     filled-in status cell for `test`/`smoke-seed`/`smoke-run`/`smoke-golden`.
  2. All 13 `core/ADAPTER-CONTRACT.md` capability names have a status in
     `.spine/capabilities.json` (hardcoded name list — stack-blind, these
     are capability identifiers, not tools).
  3. Each of six fixed foundational categories (state-management,
     persistence, module-boundaries, error-handling, auth-model,
     repo-topology — the build prompt's own list, hardcoded here as a
     disclosed, fixed checklist rather than an inferred/open set) is
     covered by an `adopted`-or-`implemented` decision carrying that
     `- Category:`, or a `docs/decisions/DEFERRED.md` entry carrying it
     with a non-empty `- Trigger:`.
  4. Adopted-decision count ≤ `--cap` (default 12). Decisions already
     `implemented` don't count against the cap — it governs design-time
     decision load, not a total that grows forever as code ships.
  `--project <path>` and `--cap <n>` are the only flags.
- **`core/templates/DEFERRED.md`** — one `## Deferred N` item per
  undecided category, `- Category:`/`- Item:`/`- Trigger:` fields, same
  `Category:` vocabulary as `decision.md`'s own field (exact-match, not
  inferred — this is what makes design-gate check #3 mechanical rather
  than fuzzy).
- **`core/templates/milestone.md`** — member tasks (ordered), inter-task
  contracts, capability targets (required content for M0), done-definition.
  Milestone IDs are `M0`, `M1`, ... — deliberately distinct in shape from
  task IDs (`<YYYYMMDD>-<slug>`) so `ledger scan-untracked-ratio`'s
  commit-trailer grep is never confused by a milestone folder (milestones
  don't commit; their member tasks do, each with its own ordinary
  `Spine-Task:` trailer). **This is Phase B's answer to open question
  §5.2** (milestone ID scheme) — recorded here for Phase E to defend or
  revise, not re-litigated.

### Extended files

- **`core/templates/decision.md`** — rewritten. Unified ID/filename scheme
  for *both* entry paths: `docs/decisions/D-<seq>-<kebab-slug>.md`
  (previously only the post-code `/ship` path existed, dated
  `<yyyy-mm-dd>-<slug>.md` — no real decision record exists yet anywhere
  real, confirmed by checking `~/horizon/docs/decisions/` and
  `~/bgr/docs/decisions/` in Phase A, so this rename carries zero
  migration cost). New header fields, each on its own `- Field: value`
  line (matching `deviations.md`'s existing exact-match discipline, and
  making `decision-hash`'s status-line exclusion and `design-gate`'s
  greps both trivial): `Id`, `Status` (proposed→adopted→implemented→
  superseded), `Category` (the fixed six, +`other`), `Source`, `Scope`,
  `Supersedes`/`Superseded-by`. New body sections: `Alternatives
  rejected` (required with reasons for `/design`; `n/a — distilled from a
  resolved deviation` is the sanctioned value for the `/ship` path),
  `Implementing paths` (append-only, `/ship`-populated), `Contracts
  implied` (optional, Extension B territory, "none" for everything this
  phase touched). Docstring rewritten to describe both entry paths
  explicitly — the old docstring's "never written speculatively" line is
  gone; the new one states which path may write speculatively (`/design`)
  and which may not (`/ship`, unchanged).
- **`core/templates/research.md`** — header gains optional
  `grounding-decisions:` block, same two-space-dash-space list shape as
  the existing `files:` block, `<id>@<hash>` entries. Explicitly optional
  — its total absence (no line at all) is a no-op, unlike `files:` which
  is required and errors if empty.
- **`core/templates/plan.md`** — new optional `## Grounds on decisions`
  section, same machine-parsed bullet convention `## Predicted touch`
  already uses (`conformance` doesn't read this section; `/ship`'s new
  decision-lifecycle step does).
- **`core/scripts/check-stale`** — new optional `grounding-decisions:`
  parse (same awk-state-machine shape as the existing `files:` parse).
  Two independent drift branches per cited decision, either sufficient
  alone: (a) `docs/decisions/<id>-*.md` doesn't resolve to exactly one
  file, (b) its current content hash (via `decision-hash`) no longer
  matches the cited one, or (c) its current `- Status:` line reads
  `superseded`. A pure status flip between any other two states
  (`adopted`→`implemented`) does **not** trigger drift — verified for
  real, see §2 below. Absence of the whole `grounding-decisions:` block is
  a total no-op — verified implicitly (every test in §2 that used it also
  ran the pre-existing `files:`-only path correctly).
- **`core/scripts/verdict-filter`** — new `decision` evidence kind:
  `{"kind":"decision","decision_id":"D-<n>","quote":"<verbatim span>"}`.
  Restructured into two passes: pass 1 (unchanged jq shape logic, extended
  with the new kind's shape check — non-empty `decision_id`/`quote`) → 
  pass 2 (new: a bash loop over pass 1's `kept` array; for `kind ==
  "decision"` entries only, resolve `docs/decisions/<id>-*.md` to exactly
  one file and `grep -qF` the quote in it — anything that fails either
  moves from kept to dropped). `file_line`/`command` verdicts skip pass 2
  entirely, unchanged behavior. New `--project <path>` flag (default pwd)
  for resolving the decisions directory, matching `check-stale`'s existing
  convention.
- **`core/skills/ship/SKILL.md` §2** (renamed "Distill decisions" →
  "Decisions", two sub-parts): **(a) new** — for each `D-<seq>` in the
  shipped plan's `## Grounds on decisions`, append this task's *actual*
  diff paths to that decision's `## Implementing paths` and flip
  `adopted`→`implemented` (leave `implemented`/`superseded` records
  alone, appending paths only). **(b) existing distillation path,
  extended** — filename convention changed to `D-<n>-<slug>.md`
  (next-id allocation documented as a one-line shell command, no new
  script — mirrors how task-ID slug generation is also skill-prose, not
  scripted), and a ship-distilled decision is now written directly as
  `implemented` (skipping the `adopted` intermediate, since the code that
  prompted it is this task's own diff) with `Implementing paths`
  pre-filled and `Alternatives rejected` defaulting to the sanctioned
  `n/a` value unless the deviation's own `Options considered` field had
  real alternatives.
- **`core/ADAPTER-CONTRACT.md`**: **not touched.** Capability status
  vocabulary (`implemented`/`unavailable`/`not-applicable`) is unchanged —
  design-time capability planning (build prompt §2's "adapters for
  code-independent capabilities are generated... at design time") uses the
  *existing* three-value schema, just populated earlier than usual. No new
  artifact type, exactly per the build prompt's own "zero new artifact
  types" constraint for Extension A.

## 2. Real invocations — what was actually run, and what it proved

All demonstrated in `~/bgr` (real spine install, `docs/decisions/` was
empty before and after — confirmed via `git status --short` both times).
Every fixture created was deleted at the end of this session; `git status
--short` in `~/bgr` is clean. Two real decision records were written
matching bgr's actual charter content (NULL≠ZERO persistence semantics;
Layer1/Layer2 module independence) rather than arbitrary placeholder text,
so the demonstration exercises the mechanism against a plausible real
decision, not a content-free stub.

1. **`decision-hash` + `check-stale`'s clean-pass path**: wrote
   `D-1-null-neq-zero-review-scores.md` (`Status: adopted`, `Category:
   persistence`), computed its hash, wrote a `research.md` citing
   `D-1@<hash>` alongside a real tracked `files:` entry (`CLAUDE.md`).
   `check-stale` reported `ok: ... (1 grounding files, 1 grounding
   decisions unchanged)`, exit 0.
2. **Content-hash drift**: edited D-1's `## Decision` body (appended one
   sentence), re-ran `check-stale` against the *same*, unmodified
   research.md (still citing the pre-edit hash). Result:
   `STALE — ... D-1 (content hash changed: cited 5013...126, now
   a351...f67)`, quarantine banner written into the research.md, exit 1 —
   confirmed the banner text is actually present via `grep`.
3. **Status-only edit does NOT drift** (the entire point of excluding the
   status line from the hash): reverted D-1's body to its original text
   (hash returned to `5013...126`, confirmed), flipped `- Status: adopted`
   → `- Status: implemented` (simulating `/ship`'s own lifecycle edit)
   against a *fresh* research.md citing the same original hash.
   `check-stale` reported `ok: ...`, exit 0 — the status flip was
   correctly invisible to the hash comparison.
4. **`superseded` branch, independent of the hash branch**: used D-2
   (`Category: module-boundaries`) instead, to isolate this from D-1's
   already-exercised hash test. Research.md citing D-2 at its correct
   hash passed clean first (`ok: ...`); flipped D-2's `- Status: adopted`
   → `- Status: superseded` (content otherwise untouched, hash therefore
   unchanged) and re-ran the *same* research.md: `STALE — ... D-2
   (status: superseded)`, exit 1 — proves the `superseded` branch fires
   independently of content drift, exactly the "either sufficient alone"
   design.
5. **`verdict-filter`'s `decision:` kind, kept and dropped, all four
   evidence shapes in one run**: a real verdicts JSON with (a) a
   `decision` verdict citing D-1 with a quote copied verbatim from its
   actual (reverted) body — **kept**; (b) a `decision` verdict citing D-1
   with a fabricated quote not present in the file — **dropped**; (c) a
   `decision` verdict citing a nonexistent `D-99` — **dropped**; (d) an
   ordinary `file_line` verdict, unrelated — **kept, unaffected**. Output:
   `dropped 2 non-conforming verdict(s)`, `kept 2/4 verdicts`, and the
   filtered JSON on disk contains exactly the two that should have
   survived — inspected directly, not just trusted from the summary line.
6. **`design-gate`, both a real FAIL and a real PASS against the same
   project**: run #1 (no `work/M0/`, no `DEFERRED.md`, D-1 at
   `implemented`/persistence, D-2 at `superseded`/module-boundaries from
   test 4 above) reported FAIL on check 1 (no milestone) and check 3 for
   five of six categories — **correctly did not flag `persistence`**
   (D-1's `implemented` status covers it) **and correctly did flag
   `module-boundaries`** (D-2 no longer counts once superseded — a bonus
   proof beyond what was originally planned, that check 3 and check-stale's
   superseded logic agree with each other). Checks 2 and 4 passed silently
   the whole time — bgr's real `.spine/capabilities.json` (all 13
   capabilities already have a status from the original `/adopt`, unrelated
   to this build) and a low adopted-count both satisfied their checks for
   free, which is itself a small proof that check 2 doesn't care *when* a
   capability's status was set, only that it exists. Wrote a real
   `work/M0/milestone.md` (using bgr's actual `smoke-*`/`test` statuses
   from its live `capabilities.json`, not invented ones) and a real
   `DEFERRED.md` covering the remaining five categories with concrete,
   bgr-specific triggers. Run #2: `PASS`. Run #3 (`--cap 0` after flipping
   D-1 back to `adopted`): `FAIL — 1 decisions are adopted, exceeding the
   cap of 0` — proves check 4 fires, not just checks 1–3.

## 3. Design decisions made this phase, for Phase E to defend or revise

- **Unified decision ID/filename scheme** (`D-<seq>-<slug>.md` for both
  entry paths) rather than keeping the old date-based scheme for
  `/ship`-distilled records and a new `D-`-scheme only for `/design`.
  Chosen because no real decision record exists anywhere yet (confirmed
  in Phase A), so there was no migration cost, and two ID schemes in one
  store would have reintroduced exactly the "one store, two formats"
  failure mode the build prompt names as this extension's central risk —
  just at the ID-scheme layer instead of the document-format layer.
- **Decision-hash excludes exactly the `- Status:` line**, resolving open
  question §5.1. Verified for real (§2, tests 2–3): a content edit drifts,
  a status-only edit doesn't.
- **Milestone IDs are `M0`, `M1`, ...** (open question §5.2, partially —
  this only answers the ID *scheme*, not its relation to task IDs beyond
  "member tasks are ordinary task IDs listed inside the milestone file,"
  which was already implied by the build prompt's own description).
- **The six foundational categories are a fixed, hardcoded list**
  (state-management, persistence, module-boundaries, error-handling,
  auth-model, repo-topology) rather than something `/design` infers or
  negotiates per project. This is the build prompt's own example list,
  taken literally as the checklist rather than as illustrative — flag for
  Phase C/E: if a real greenfield project's design stage finds this list
  wrong for its shape (e.g. a CLI tool with no auth model at all), the
  fix is a `Category: other` escape hatch (already present) plus a
  disclosed limitation, not a silent category skip. Not yet tested against
  a real `/design` run — Phase C's worked example is where this gets
  pressure-tested for real.
- **Ship-distilled decisions skip the `adopted` intermediate and land
  directly at `implemented`.** This wasn't explicitly specified by the
  build prompt (which describes the lifecycle from `/design`'s point of
  view) but follows directly from its own logic: the code that prompted a
  ship-distilled decision already exists as that task's diff, so there is
  no real window where the decision is "adopted but not yet implemented."
  Recorded as a Phase B interpretation, not a re-litigation — flag if
  Phase E's tradeoffs review disagrees.

## 4. What the next session needs that isn't obvious from the files

- **Phase C is next**: `/design` (the interactive skill that actually
  produces `docs/charter.md`, `docs/decisions/D-*.md` at `proposed`→
  `adopted`, `DEFERRED.md`, `work/M0/milestone.md` for real, on a genuine
  new project), the adversary design-mode mandate sections in
  `core/agents/falsifier.md`/`security.md` (neither read in full yet —
  Phase A only confirmed the verdict schema they already produce), and
  `/task`'s `--milestone <id>` argument. Then the greenfield worked
  example end to end, per the build prompt's own Phase C scope.
- **A real project for the greenfield worked example has not been
  chosen.** Per Phase A's own note, this should be an `AskUserQuestion` at
  the start of Phase C, not decided unilaterally here.
- **The category-checklist risk above is unresolved until Phase C runs it
  for real** — this phase only proved the mechanism is scriptable and
  correct against a fixed, hand-authored input; it did not prove the
  six-category list is the right list for an arbitrary greenfield project.
- **`core/scripts/decision-hash` currently has no `--self-test`.** Every
  other `core/scripts/*` in this repo is a plain deterministic utility
  without the adapter self-test convention (that convention is
  `.spine/adapters/*`-specific per `ADAPTER-CONTRACT.md` §4, project-side,
  not core-side) — `decision-hash`/`design-gate`/`check-stale`/
  `verdict-filter` are all core scripts, same category as the pre-existing
  `check-stale`/`verdict-filter`/`floor`, none of which carry `--self-test`
  either. Consistent with existing precedent, not an oversight, but noting
  it explicitly in case Phase E's self-red-team wants to name it as a
  residual (a core script bug in `decision-hash` itself would only be
  caught by exactly this kind of ad hoc real-invocation testing, never by
  an automated conformance suite the way a project adapter's bug would be).
- **No spine core file was modified this phase beyond what's listed in
  §1.** `git status --short` in `~/spine` before writing this handoff
  shows exactly those files as `M`/`??`, plus the pre-existing Phase-E
  changes already flagged as unrelated in `ext-phase-A-handoff.md`.
