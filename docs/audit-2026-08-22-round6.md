# Audit — 2026-08-22, round 6

Independent, adversarial audit run against `main`. Governance check (every
round's own opener): confirmed commit `af18909` ("test: add core-selftest
coverage for propagate, setup, ledger-field-audit, render-dashboard") is an
ancestor of `HEAD` via `git merge-base --is-ancestor` — and `HEAD` **is**
`af18909`. Four commits landed after round 5's own report was finalized
(baseline `92b7af5`): `7e178f8` (round 5's own code fixes — the hooks
`segs[@]` crash, two fuzz-harness `set -e` bugs, and 11 dead-citation
fixes) and three more (`85a7eb8`, `45b3103`, `7d51375`, `af18909`) that
added permanent `core-selftest` coverage for all 17 previously-uncovered
`core/scripts/*` scripts. That coverage work predates no audit report —
this round is its first independent scrutiny, treated as this round's
primary object of study per the brief, not as already-audited history.

This round's own repo-wide diff since round 4's last full-depth baseline
(`92b7af5`) is exactly: `core/hooks/_workspace-route` (the `segs[@]`
crash fix), `core/scripts/core-selftest` (the ~2,000-line coverage
addition), and 9 files touched by round 5's citation-hygiene sweep
(`core/ADAPTER-CONTRACT.md`, `core/rules/contracts.md`,
`core/skills/ship/SKILL.md`, `core/skills/tasks/SKILL.md`,
`core/templates/plan.md`, `core/templates/workspace.json`,
`core/scripts/{claims-check,contract-touch,registry-sync}`). Every other
file in the repo — `core/agents/*`, `core/rules/migrations.md`,
`README.md`, `docs/tradeoffs.md`, and every `core/skills/*` file not in
the list above — has **zero diff** since round 4's full-depth pass.
Confirmed directly via `git diff --stat 92b7af5 af18909`, not assumed.

Two new fixes were made and applied directly during this round (see
Priority 2), following the established norm of every prior round. As with
round 3's own governance note: **these fixes are left uncommitted**,
consistent with this project's own norm of not auto-committing audit
fixes — for the human running this audit to review and commit.

---

## Priority 1 — adversarial verification of the new `core-selftest` coverage

This is genuinely unaudited surface: 17 scripts, ~140 new cases, added in
the four commits after round 5's report was written. Verified by mutation
testing, not by reading the tests and trusting they'd catch a break —
matching every prior round's own standing lesson ("a script is trusted
once it's been run against a case designed to break it, never before").

### The 12 scripts with no stated negative control — each mutated directly

For each, the highest-consequence real assertion was identified, the
**real script** (not the test) was mutated to break exactly that
property, `core-selftest` was re-run in full, the failure was confirmed,
and the script was restored via `git checkout` before moving to the next.
`git status` was clean between every mutation.

| Script | Mutation | Result |
|---|---|---|
| `decision-hash` | Disabled the `- Status:` line exclusion (`grep -v` → `cat`) | **FAIL as expected** — status-only-change case caught it |
| `profile-check` | Added `"floor"` to the allowed-keys list (the exact smuggled-bypass scenario the check exists to catch) | **FAIL as expected** |
| `q` | Changed `exit $code` → `exit 0` | **FAIL as expected** — exit-code passthrough case caught it |
| `ui-touch` | Broke the glob `case` match (`$glob` → a literal that never matches) | **FAIL as expected** |
| `verdict-filter` | Forced the decision-quote containment check to always pass (`if true`) | **FAIL as expected** — fabricated-citation case caught it |
| `next-milestone-task` | Disabled the "predecessor not done" gate (`if false`) | **FAIL as expected** — BLOCKED case caught it |
| `design-gate` | Made `decision_covers()` always return success | **FAIL as expected** — uncovered-category case caught it |
| `decision-index` | Changed `sort -n` to lexical `sort` | **FAIL as expected** — D-9-before-D-10 case caught it |
| `propagate` | Made the `implement/verify/ship` branch report `advisory` instead of `disruptive` (the under-signaling direction) | **FAIL as expected** |
| `setup` | Removed the strict-mode pin-mismatch block condition | **FAIL as expected** |
| `ledger-field-audit` | Disabled the zero-hits detection branch (`if false`) | **FAIL as expected** — dead-field case caught it |
| `render-dashboard` | Redacted the Gantt-bar visible label | **PASS unchanged** — see below |

11 of 12 mutations were caught cleanly and immediately by the existing
coverage. **`render-dashboard`'s single case deserves an honest caveat,
not a pass/fail label alone**: mutating the human-visible Gantt label
alone didn't fail the smoke test, because the task id is independently
embedded in the same HTML via the `href` and `data-*` attributes the test
never specifically asserts on — the test's actual property is "renders
without crashing and the task id appears somewhere," which held true. A
follow-up mutation that introduced a genuine crash trigger (an unbound
variable reference under this script's own `set -u`) **did** fail the
test correctly (exit 1, caught). A third check — calling an undefined
function inside a `$(...)` substitution — did **not** crash the script,
because `render-dashboard` has no `set -e`: a failed command substitution
inside a subshell doesn't propagate under `-uo pipefail` alone. This
means the smoke test reliably catches `nounset`-class breaks but not a
broken helper call whose failure is silently absorbed by a subshell —
consistent with, not contradicting, the section's own header comment
("smoke coverage only... confirms it runs without crashing and produces
HTML containing what it claims to" — never claimed per-field rendering
correctness). Not a false claim; a real, disclosed, and now independently
confirmed scope limit.

### The 5 scripts with a stated negative control — re-verified with a different mutation on the same property

Per the brief: re-ran one of round 5's own negative controls directly
(all 5 already re-run as part of every full `core-selftest` pass above —
all green), then applied a **second, independently-designed mutation**
targeting the same underlying property through a different code path.

| Script | Round 5's own mutation | This round's independent mutation | Result |
|---|---|---|---|
| `check-stale` | Reverted the status-at-citation comparison to literal `"superseded"` | Broke the `git show`-based historical-status lookup itself (`status_at_citation="BROKEN_LOOKUP..."`) | **FAIL as expected** — Case F (already-superseded-at-citation) caught it |
| `registry-sync` | Changed the scoped `git add --` to `git add -A` | Left the real scoped add alone but appended an *extra* `git add -- "src/"` call (a "someone helpfully broadened the sweep" regression shape) | **FAIL as expected** — Case C (scope discipline) caught it |
| `second-approver-check` | Inverted `== ` to `!=` in the identity comparison | Broke the owner-file reader instead (`owner="$(cat ...)"` → a fixed placeholder) | **FAIL as expected** on the primary self-approval case. (The negative-control sub-test at the end of the same run also failed, but that's an artifact of stacking two mutations inside one run — the negctrl's own backup/restore captured *my* already-mutated file as its baseline, not the original; not a real coverage gap.) |
| `contract-touch` | Disabled the breaking-vs-additive awk heuristic | Disabled `registry_stale` detection instead (a different property in the same script) | **FAIL as expected** — Case D (stale registry) caught it |
| `claims-check` | Turned write/write blocking into a warning | Disabled the `done`-state task exclusion instead (`[[ ... == "done" ]]` → `false`) | **FAIL as expected** — Case D (done-task exclusion) caught it |

Priority-1's own load-bearing pair (`check-stale`'s already-superseded
guard, `registry-sync`'s scope discipline) both held under a genuinely
different failure mode than round 5 tested, not just a re-run of the same
mutation.

### Fixture realism — spot-checked against real installed-project data

`claims.json`'s field shape (`predicted_touch`, `grounding_files`,
`grounding_decisions`, `contracts`) matches real `tgml` claims.json files
exactly, field-for-field. `.spine/ui-paths.conf`'s glob-per-line format
with `#`-comment support matches `tgml`'s real file byte-for-byte in
convention. One real divergence found: synthetic `owner`/`approval.json`
fixtures in `core-selftest` use single-token identities (`"alice"`,
`"bob"`), while every real `owner`/`approval.json` file across all
installed projects uses a two-token `"name email"` string (e.g.
`"michaelbart michaeljbart@me.com"`) — the coverage never exercises an
identity string containing a space. Directly tested this shape against
the real `second-approver-check` script by hand: works correctly (bash
string comparison doesn't care about embedded spaces), so this is a real
but non-consequential fixture-realism gap, not a bug — noted rather than
silently absorbed. Real decision files (`docs/decisions/D-*.md`) also use
a different field order/set (`Id, Status, Category, Date, Source, Scope,
...`) than the synthetic fixtures (`Id, Category, Status, Scope, ...`);
non-consequential since every consuming script (`design-gate`,
`decision-index`, `decision-hash`) parses each field with its own
line-anchored regex, order-independent — confirmed by inspection, not
assumed.

### Full suite, run cold

`bash core/scripts/core-selftest`: **140 PASS, 0 FAIL**, three independent
runs during this round, all clean. Wall time: 60.5–61.2s across runs
(`16.3s user + 29.9s system`) — slightly over the "~55–60s" figure named
in this round's own brief, up from round 5's measured 47.7s. Not a
regression: the increase tracks the ~140 additional cases added across
the four post-round-5 commits, matches expectations, and stays well
within any operationally meaningful bound (no `timeout`/`gtimeout`-style
hang, still fork-bounded per the fuzz harness's own `fuzz_run_bounded`).

### Priority 1's own question, answered directly

**Did adversarial re-verification change confidence that this session's
new `core-selftest` coverage constitutes real regression protection?**
Yes — materially increased, not decreased. 16 of 17 targeted mutations
(12 fresh scripts, 5 with independently-varied mutations) were caught
immediately and correctly; the one exception (`render-dashboard`) failed
only on a mutation outside its own explicitly disclosed scope, and
correctly caught a mutation inside that scope. No vacuous case was found
— every "PASS" this round confirmed was genuinely load-bearing, not a
tautology. **What would make this trust the wrong call going forward:**
if a future change to one of these 17 scripts' real logic ships without
re-running `core-selftest` (the same discipline every prior round has
asked for), or if a new script is added to this coverage without a
negative control and nobody re-derives one adversarially the way this
round did — the same standing risk this report itself exists to catch,
not a defect already found.

---

## Priority 2 — citation/dead-reference hygiene

### `core/hooks/*` — light diff-only, per the classification table's own instruction

Confirmed **zero diff** in `core/hooks/*` since `7e178f8` (`git log
--oneline -1 -- core/hooks/` shows only `7e178f8`). Per round 5's own
graduation condition ("stepping down is wrong if a future change ships
without a fresh `core-selftest` run"), that condition wasn't tested this
round because nothing changed — so it stays untested, not passed. Re-ran
the fuzz section as a live confirmation anyway, per this round's own
brief (cheap, ~confirmed as part of every full-suite run above): all four
fuzz sections (`_wsr_collapse`, `_wsr_realpath`, `workspace_route`,
`_wsr_glob_eq`) still pass clean, 4,620+ generated cases. **Verdict:
stays at light-touch for round 7, still conditional** — this is now 2
rounds of zero-diff confirmation (rounds 5 and 6) plus 1 round of real,
from-scratch fuzz-discovery (round 5's live run). The condition for
stepping back up (a future change shipping without a fresh
`core-selftest` run recorded) has not fired.

### The "build prompt §N" / "open question §N" bug class — swept for the deleted document, not one more phrase

Per round 5's own explicit instruction — this round swept for every
document deleted in commit `05e4886` (which removed all of `work/`,
including `work/.build/*-handoff.md` and `docs/proposals/*.md`), not for
one more named citation phrase.

**Two new, genuine, previously-unswept dead citations found and fixed:**

1. **`core/scripts/contract-touch:18`** — `"...bookmarks' own
   callers/lint adapters, per ext-phase-C-handoff.md §2)."` cited
   `work/.build/ext-phase-C-handoff.md`, deleted in `05e4886`. **Fixed:**
   dropped the dead citation; the sentence's substantive claim (the bash
   3.2 limitation was already fixed the same way elsewhere) reads
   identically without it.
2. **`core/templates/pr-description.md:33`** — `"...it fires on this
   patch's own demonstration record (see the Phase B handoff's trace
   audit)."` cited one of `work/.build/{phase-B,ext-phase-B,ext-c-phase-B,
   pr-patch-phase-B}-handoff.md`, all deleted in `05e4886`. **Fixed:**
   reworded to state the underlying fact directly ("it has fired on real
   overlapping file citations before") without pointing at a document
   nobody can open. Both fixes are comment/prose-only — confirmed via
   `git diff` (shown below) and a clean, unchanged `core-selftest` run
   (140 PASS / 0 FAIL) immediately after.

**A related, much larger pattern found and deliberately left as-is,
not fixed:** 23 live citations to `"Extension C §N.N"` / `"Extension C
Phase N"` across 10 files (`core/scripts/ledger`, `setup`,
`second-approver-check`, `render-dashboard`; `core/templates/hook-guard`;
`core/skills/costs`, `bootstrap`, `workspace`, `task`, `ship`). "Extension
C" is almost certainly the same deleted `work/.build/ext-c-phase-*.md` /
`ext-phase-*.md` handoff-document family — no live document anywhere in
the repo defines or explains what "Extension C" is (grepped for a
definition; found none). **This is not the same failure shape as the
citations fixed above.** Every one of the 23 instances checked states its
operational rule directly, in the same sentence, with the citation
functioning as a parenthetical provenance/version tag rather than a
"go read this section to learn the rule" pointer — e.g. `## 0. Ship-time
re-grounding (Extension C §2.4)` is a section heading whose entire body
explains the rule; `"the registry is shared state" (Extension C §2.2)`
states the fact before the tag. A reader never hits a dead end trying to
resolve one of these. This matches round 4's own precedent for
`ADAPTER-CONTRACT.md`'s "callers incident" narrative (kept, not fixed,
because it's self-contained history, not a broken pointer) — extended
here to a much larger, still-self-contained set. **Verdict: swept,
confirmed non-broken, left alone per "no changes for the sake of
changes."** Flagging this explicitly rather than silently passing it by,
since the sheer count (23, the largest citation-pattern found in any
round to date) makes it worth a maintainer's own judgment call on whether
historical-provenance tags to a permanently-unresolvable name are worth
keeping — but that's a judgment call, not a defect this audit's own
mandate (fix broken references) covers.

Confirmed via a wrap-tolerant, case-insensitive sweep (joins lines before
matching, so a phrase split across a line wrap — the exact reason 3 of
the last 4 rounds' sweeps missed things — can't hide) across every
`.md`/`core/scripts/*`/`core/hooks/*` file for: every deleted filename
stem (`build-prompt-extensions`, all 15 `*-handoff.md` variants,
`phase-A/B/C/D/E`), `g1-tee-waitlist` (round 1's original finding, still
clean), `two-engineer-demo` and `docs/example` (the deleted demo
directory), and all three `docs/proposals/*.md` titles/keywords
(`carryforward`, `adaptive-autonomy`, `task-visualization`) — zero
further live hits beyond the two fixed above.

### Diff-touched files from round 5's own citation fix — meaning preserved, edits comment/prose-only

Read the full `git show 7e178f8` diff for all 9 touched files
(`core/ADAPTER-CONTRACT.md`, `core/rules/contracts.md`,
`core/skills/ship/SKILL.md`, `core/skills/tasks/SKILL.md`,
`core/templates/plan.md`, `core/templates/workspace.json`,
`core/scripts/{claims-check,contract-touch,registry-sync}`) directly,
line by line, not summarized from the commit message. Every edit falls
into one of two shapes, both preserving the exact prior operational rule:

- **7 of 9 files**: pure citation drop, rule text otherwise byte-for-byte
  unchanged (`ADAPTER-CONTRACT.md` ×2, `contracts.md`,
  `registry-sync`, `ship/SKILL.md`, `tasks/SKILL.md`, `plan.md`,
  `workspace.json`).
- **2 of 9 files** (`claims-check`, `contract-touch`): the dead citation
  was replaced with an inline paraphrase of what it pointed at (e.g.
  `"the race named in open question 7"` → `"the race between two
  engineers reading/writing the shared registry concurrently"`) —
  verified each paraphrase against the surrounding code's actual
  behavior (claims-check's pull-first race, contract-touch's
  conservative-gate-anyway posture) and confirmed accurate, not just
  plausible-sounding.

All three touched scripts' diffs (`claims-check`, `contract-touch`,
`registry-sync`) are **confirmed comment-only** — every changed line in
each diff hunk begins with `#`, zero executable-line changes, verified by
reading the raw diff directly rather than trusting the commit message's
own "comment-only" claim.

---

## Priority 3 — light diff-only spot-checks

Every item below has **zero diff** since its own last full-depth audit,
confirmed via `git diff --stat 92b7af5 af18909` (the complete file list
that changed anywhere in the repo since round 4's baseline — reproduced
in this report's opening section). "Zero diff" here is the strong form
round 5 used for its own Priority 3: not "nothing looked wrong," but
"provably nothing could have drifted."

- **`core/agents/*`** — zero diff since round 4's full-depth pass. This
  is now round 5 *and* round 6 both confirming zero diff after round 4's
  real, from-scratch clean check — **3 consecutive rounds, the third
  under the classification table's own graduation bar** (round 4 real
  full-depth clean; rounds 5–6 zero-diff confirmations). Per the table's
  own stated condition ("if this round also finds nothing... can graduate
  to opportunistic-only starting round 7"): **graduates out of the
  standing rotation starting round 7.** What would pull it back: any diff
  touching `core/agents/*`, or a downstream skill changing what it claims
  an agent does/produces.
- **`core/rules/migrations.md`** — zero diff since round 2's clean check
  (round 4 re-confirmed). Stays light diff-only; not yet 3 consecutive
  real-checked clean rounds by the same strict count (rounds 3 and 5
  didn't specifically re-examine it, they inherited "no diff" transitively).
- **`README.md`** — zero diff since round 3's fix (round 4's own
  full-depth pass found it accurate). Rounds 5 and 6 are diff-only
  confirmations, not from-scratch re-checks. **1 real clean round (4) +
  2 zero-diff confirmations (5, 6)** — this is meaningfully different
  from `core/agents/*`'s 3-round count above (which had a genuine
  full-depth clean pass at round 4 too, but round 4's agents check and
  round 4's README check are not the same kind of scrutiny — the agents
  check was itself adversarial cross-referencing against every real
  invocation site; the round-4 README check was a table cross-reference).
  Stays light diff-only, not yet graduation-eligible under a strict
  reading of "real, from-scratch checking."
- **`docs/tradeoffs.md` accuracy** — zero diff since round 4's full
  accuracy pass (every disclosed limit checked against code). Same count
  as README: 1 real full pass + 2 zero-diff confirmations. Stays light
  diff-only.
- **`core/skills/*` not touched by the citation-hygiene diff** — zero
  diff since round 4's fixes (which themselves followed CONFIRMED
  findings in every one of rounds 1–4). This is the **second** consecutive
  zero-diff round (5, 6) after 4 straight rounds of real findings — stays
  light diff-only, explicitly **not yet graduation-eligible**: the
  classification table's own bar requires "at least one round of real
  full-depth re-checking with nothing found first," and neither round 5
  nor round 6 did a from-scratch re-read of this set — both relied on
  zero-diff. Round 7 should do one real full-depth pass over this set if
  it still shows zero diff, specifically to earn graduation rather than
  assume it.
- **`core/scripts/*`'s own logic** (`floor`, `ledger`, `render-task`,
  `conformance`, `adapter-conformance`, `adversary-cache-tier`, and the
  14 of 17 newly-covered scripts whose bytes never changed) — confirmed
  via the same diffstat: zero byte changes to any of these files' actual
  logic since round 4. The only `core/scripts/*` files with any diff at
  all are `core-selftest` (the coverage addition — Priority 1, above) and
  `claims-check`/`contract-touch`/`registry-sync` (comment-only citation
  edits, verified above). Stays light diff-only for the scripts' own
  logic, same as the table's starting classification.

---

## Priority 4 — continued measurement

### The mark-vs-harvest gap — re-measured a fourth time, still flat, but real new post-fix data now exists

Re-measured across all 6 installed projects (same 29 real `ledger.json`
files round 5 counted, plus 1 new one — `tgml/20260823-watchlist-
tickets-pins`, an in-progress task created after round 5's fix landed —
30 total). **22/30 (73.3%)** are missing `.phases.implement.tokens` —
essentially unchanged from round 5's 72.4% and round 3/4's ~65%. Of the
22 misses, 20 (90.9%) have `implement.marked_at` present with the harvest
specifically absent — matching round 5's 90% figure almost exactly. **The
gap has now been measured 4 times (rounds 3, 4, 5, 6) and has not moved
outside a ~65–73% band in any of them**, across a visibility fix that has
now been live for ~2h20m of real usage at measurement time (round 4's fix
landed 07:30 local, round 5's landed 08:34, this measurement taken
10:53–14:53 the same day).

**Genuine new post-fix data exists this round, for the first time.** One
real task, `tgml/20260822-filmview-verdict-menu-watchlist`, shipped
(state: `done`) with its `ship.marked_at` (13:18:20Z) landing ~44 minutes
after round 5's fix (12:34:07Z) — the first fully-completed real task
whose late-stage phases (falsifier, security, ship) postdate any fix in
this audit's history. Its own `implement.marked_at`, however, predates
round 4's fix by 13 seconds (the same in-flight-at-fix-time task round 5
already identified) — so this task still doesn't answer whether the
*harvest* behavior itself has changed; it only confirms the visibility
fix (`render-task`'s "(not recorded)" caveat) continues to render
correctly on real data, which was already established.

**A second, currently-in-progress task**
(`tgml/20260823-watchlist-tickets-pins`) has `implement.marked_at`
(13:48:42Z) that genuinely postdates round 5's fix — the first task
whose entire `implement` phase mark happened under fully-fixed code. It's
still in `verify` state as of this measurement, with `implement.tokens`
not yet harvested. **Worth a real, no-guessing check next round**: does
this specific task's harvest end up populated or missing by the time it
ships? That's the first individual, unambiguous post-fix data point this
gap will have.

**Naming the elapsed-time threshold, as this round's brief asked for
after a third flat measurement (this is now the fourth):** `tgml`'s real
task-creation rate over its own history (23 tasks across ~5 real days,
Aug 18–23) is roughly 3–5 tasks/day. A statistically meaningful sample
(the same rough size as the 20–29-task samples every prior measurement
already used) of **post-fix-only** tasks would need on the order of **10
newly-created-and-shipped tasks after 2026-08-23T12:34:07Z** — at the
observed real rate, that's **roughly 2–3 more calendar days** of ordinary
usage, not a same-session re-measurement. Round 7, if run the same day as
round 6, will almost certainly show the same flat number for the same
reason this round did — re-measuring before that many real tasks
accumulate produces noise, not signal. Recommend round 7 measure only if
it runs at least 2 calendar days after this one; otherwise it should
explicitly state it's skipping re-measurement for that reason rather than
re-running a same-day check.

### Round 4's two prose-only fixes — real post-fix data now exists, and both hold

**Both were "unanswerable until a session writes a briefing after this
morning" as of round 5.** That session now exists:
`tgml/20260822-filmview-verdict-menu-watchlist`'s `briefing.md` and
`pr-description.md` were both written after round 5's fix landed
(`ship.marked_at` 13:18:20Z, 44 minutes post-fix).

- **"One bullet per deviation, never a paragraph"** — this real task had
  2 real, formalized deviations (`deviations.md`, both `Tier: record-
  and-proceed`/`halt`, both `Status: resolved`). Both `briefing.md`'s
  "What surprised us" section and `pr-description.md`'s own "What
  surprised us" section render them as **two separate markdown bullets**,
  not a compressed run-on paragraph with parenthetical numbering — the
  exact shape the tightened round-4 wording exists to prevent, and the
  exact shape round 4's own pre-fix evidence
  (`filmview-synopsis-trailer-watch`) showed failing. **First real
  confirmation the tightened wording changes real output, not just a
  spot-check of unrelated older tasks.**
- **`pr_description_line_count`/`pr_description_word_count` firing on
  real content** — this task's `ledger.json` has both fields populated
  (`34` lines / `594` words), and both **exactly match** the real
  `wc -l`/`wc -w` output of the actual `pr-description.md` file on disk
  (verified directly, not trusted from the ledger). `render-task` renders
  correctly against this real (not synthetic) data. At 594 words, this
  instance sits just under the 600-word cap — no over-cap rendering to
  observe yet, but the field-population and exact-match mechanics are now
  confirmed against genuine real-world output for the first time, closing
  the gap round 4 disclosed as synthetic-fixture-only.

---

## What's working well

- **This round's own new `core-selftest` coverage held up under real,
  independent adversarial mutation** — 16 of 17 targeted breaks caught
  immediately, the one exception failing only outside its own disclosed
  scope. This is real, load-bearing regression protection, not
  vacuous test theater.
- **The citation-hygiene bug class, swept properly this time** (for the
  deleted document, not one more phrase) found exactly 2 new genuine
  instances after 5 rounds and 4 prior sweeps — a shrinking, converging
  problem, not a recurring one at the same rate. The much larger
  "Extension C" pattern was correctly triaged as a different, non-broken
  shape rather than mechanically "fixed" for its own sake.
- **Real, positive confirmation of two previously-synthetic-only fixes**
  (deviation-bullet formatting, `pr-description.md`'s word-cap fields) —
  both now verified against genuine post-fix real-world output, both
  holding as designed.
- **The governance discipline holds**: `HEAD` was exactly the expected
  commit, `git status` was clean at the start, and this round's own two
  fixes are comment/prose-only, verified line-by-line before being
  trusted, with a clean `core-selftest` run immediately after.

## Overall verdict

Round 6's central question — does this session's new `core-selftest`
coverage constitute real regression protection, or does it call the
batch's reliability into question — has a clear, evidence-backed answer:
**it's real.** Every mutation targeting a genuine, load-bearing property
was caught; the one script that didn't fail on one specific mutation
(`render-dashboard`) failed correctly on a mutation inside its own
disclosed scope and only survived a mutation the section's own header
never claimed to guard against. Five scripts that already had a stated
negative control held up under a second, independently-designed attack on
the same property through a different code path — this wasn't a
rubber-stamp re-run.

The citation-hygiene class, audited for real this time (against the
deleted document, not one more phrase), converged rather than recurred at
the same scale: 2 new instances found and fixed, down from 84 (round 3),
11 (round 5's own sweep), to 2. The much larger "Extension C" pattern
this round newly discovered was correctly distinguished as structurally
different (self-contained provenance tags, never a broken pointer) rather
than mechanically fixed to inflate a finding count — consistent with
"no changes for the sake of changes."

Two subsystems graduate or move meaningfully closer to graduating this
round: `core/agents/*` (3 consecutive clean rounds under the table's own
bar, explicitly graduating starting round 7) and `core/hooks/*` (stays
conditional, now 2 zero-diff rounds after round 5's real fuzz-discovery
round, still waiting on its own stated re-run-on-next-change condition).
Nothing regressed. Nothing was found broken and left unfixed.

**What would change this verdict:** if a future round's mutation testing
of any of these 17 scripts' coverage finds a case that doesn't fail under
a real break — this round found none, but only tested one mutation per
script; a more exhaustive pass (multiple mutations per script, the same
depth this round gave the 5 pre-existing-negative-control scripts) could
still surface a gap this round's single-mutation-per-script budget
didn't reach. That's the honest limit of this round's own coverage of
the coverage.

---

## For round 7

- Track `tgml/20260823-watchlist-tickets-pins` to completion — its
  `implement.marked_at` is the first real mark that postdates round 5's
  fix outright; whether its harvest ends up populated is the first
  unambiguous individual data point for the mark-vs-harvest question.
- Don't re-measure the mark-vs-harvest gap same-day; wait for ~10 newly
  created-and-shipped post-2026-08-23T12:34:07Z tasks (roughly 2–3
  calendar days at `tgml`'s observed rate) or explicitly state why
  re-measuring anyway is still worthwhile.
- `core/agents/*` graduates to opportunistic-only starting this round —
  don't budget standing rotation time against it unless a diff appears.
- `core/skills/*` (the set not touched by round 5's citation diff) is one
  real full-depth pass away from the same graduation `core/agents/*` just
  earned — worth spending that pass specifically if round 7 has budget,
  rather than defaulting to another zero-diff confirmation.
- If `core/hooks/*` changes for the first time since round 4, re-run the
  fuzz harness as part of that change's own verification — this is the
  standing condition round 5 set and round 6 didn't get to test (nothing
  changed).
- Consider raising the "Extension C §N" pattern (23 citations to a
  permanently unresolvable historical name, across 10 files) with whoever
  owns this repo as a judgment call, not a defect — worth a deliberate
  decision (keep as provenance history, or strip as this round's fixed
  citations were) rather than another round re-discovering it.
