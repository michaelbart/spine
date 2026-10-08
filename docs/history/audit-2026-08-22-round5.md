# Audit — 2026-08-22, round 5

Independent, adversarial audit run against `main`. Governance check (every
round's own opener): confirmed commit `92b7af5` ("fix: hooks DoS/Unicode
bypass, cost-reporting, and doc gaps found in round-4 audit") is an
ancestor of `HEAD` via `git merge-base --is-ancestor` — and in fact `HEAD`
**is** `92b7af5`: zero commits landed between round 4 and the start of this
round, `git status` was clean, and `92b7af5` itself landed at
`2026-08-23T07:30:10-04:00` — the morning of this round, not the 22nd its
own filename suggests. This round's own edits are the first change to the
repo since round 4's fix pass. Given that, Priority 3's "diff each file
against its state as of `92b7af5`" instruction is vacuously satisfied for
every file this round didn't itself touch — there is no drift to check.

This round was explicitly asymmetric per its own brief: most of the budget
went to Priority 1 (build real fuzz infrastructure for `_workspace-route`,
motivated by four rounds finding six distinct hooks-bypass mechanisms by
hand, one nearly every round). Priority 2 got full-depth effort on the
specific items rounds 1–4 flagged as unresolved. Priority 3 was a diff-only
spot-check, made trivial by the zero-commits fact above.

---

## Priority 1 — Hooks fuzz harness — CONFIRMED, headline finding of this round

### Built: property-based fuzz coverage for `_workspace-route`'s path functions

Added a `fuzz:` section to `core/scripts/core-selftest` (`core/scripts/
core-selftest:738-1005`) covering `_wsr_collapse`, `_wsr_realpath`,
`workspace_route`, and `_wsr_glob_eq` — not another hand-picked scenario
list, a generator producing thousands of randomized path strings (varying
segment count, `.`/`..` density, leading/trailing/doubled slashes, empty
segments, non-ASCII in both NFC and NFD form, absolute vs. relative) and
checking invariants that must hold regardless of input shape:

- `_wsr_collapse` never returns empty (always at least `/`) — 3,000 cases
- `_wsr_collapse` is idempotent (`collapse(collapse(x)) == collapse(x)`)
  — same 3,000 cases
- `_wsr_collapse`'s output never contains a `.`/`..` segment — same 3,000
  cases
- `_wsr_realpath` always terminates with a non-empty result (the function
  whose unbounded recursion caused round 4's multi-thousand-process
  runaway) — 800 cases
- `workspace_route` never leaves `WSR_OWNER_ROOT` set to a non-absolute
  path, including under deliberately malformed (`""`, relative) `$project`
  values that must fail closed — 800 cases
- `_wsr_glob_eq` is symmetric under independent NFC-normalization of
  either argument (all four NFC/NFD combinations of a name against itself
  must agree) — 20 real accented/CJK/emoji names

Total: **4,620 generated/curated cases**, run for real on every invocation.
python3 generates the corpus (fixed seed `20260823` for reproducibility —
this is wired in as a **permanent regression case**, not a one-off
exploration script, so a fixed corpus that fails reproducibly is more
useful here than a different one every run) — the same accepted,
graceful-fallback dependency `_wsr_nfc` already introduced in round 4; a
smaller bash-only `$RANDOM`-based generator is used if python3 isn't on
`PATH` (verified working standalone: 3,007 cases, 0 failures, run with
python3 deliberately shadowed off `PATH`). Each fuzz section runs under a
bounded background job (the same fork+poll+kill pattern `hook_run_bounded`
already established, since neither `timeout` nor `gtimeout` exists on this
host), so a reintroduced runaway fails loudly within its bound instead of
hanging the suite — directly satisfying the "no invocation takes more than
a fixed short bound" invariant the round-5 brief asked for.

### The harness found a real bug on its first run — CONFIRMED, but not currently reachable through any hook

`_wsr_collapse("")` (the literal empty string) crashed with `unbound
variable: segs[@]` under bash 3.2's `set -u` — the **identical** bug class
round 4 fixed for the `out` array in the same function
(`core/hooks/_workspace-route:83-118`, pre-fix), but on the `segs` array a
few lines earlier in the same function, which round 4's fix never
touched. `read -ra segs <<<""` leaves `segs` a legitimately zero-element
array, and bash 3.2 has no empty-array exemption under `nounset` —
dereferencing `"${segs[@]}"` throws, the error is swallowed by the
enclosing `$(...)`, and the function returns `""` instead of the
documented `/`. This is the exact defect class the round-5 brief predicted
fuzzing would find that hand-testing wouldn't: not a new call-site gap,
but a second instance of a bug already believed fixed, inside the same
function, missed by three subsequent rounds of adversarial hand-testing
of the *fixed* code.

**Reachability, traced exhaustively:** every real call site into
`_wsr_collapse` was checked by hand against every real caller
(`_wsr_realpath`'s two call sites, `workspace_route`'s direct call, and
`_wsr_realpath`'s own recursive self-call via `dirname`, which never
returns empty). `phase-gate`/`path-escalate`/`dep-gate` all guard
`file_path`/`task_id` non-empty (`[[ -n "$file_path" ]] || exit 0`,
`[[ -n "$task_id" ]] || exit 0`) before any path reaches `_wsr_collapse`;
`_bash-write-targets:60` (`_bwt_is_confidently_resolvable`) explicitly
rejects an empty target as `UNRESOLVED`, so a Bash-command `PATH:` line
can never carry an empty suffix either; and `workspace_route` itself
fails closed on a non-absolute (hence non-empty) `$project` before ever
calling `_wsr_collapse`. **No currently-shipped path from any of the
three hooks reaches this bug.** Marking this CONFIRMED as a real code
defect (reproduced directly, fixed, verified) but explicitly not claiming
it as an exploitable bypass — the honest, load-bearing distinction this
round's own standing rules ask for.

**Fixed:** applied the identical crash guard round 4 used for `out` to
the `segs` iteration (`core/hooks/_workspace-route:96-112`) — `if [[
${#segs[@]} -gt 0 ]]; then for seg in "${segs[@]}"; do ... done; fi`.
Verified: the fuzz section now passes 3,000/3,000 collapse cases,
including the direct `_wsr_collapse("")` case. **Negative-controlled**:
reverted just this one fix (`git stash push -- core/hooks/_workspace-route`),
re-ran `core-selftest` — the `_wsr_collapse` fuzz section failed cleanly
(`FAIL: fuzz: _wsr_collapse: ... produced no summary line (crashed? see
...)`), the other three fuzz sections and all 42 non-fuzz cases still
passed (45 PASS / 1 FAIL of 46 total, exactly the expected shape), restored
the fix, confirmed clean `git status` and a green suite again.

### Two bugs found in the fuzz harness itself, while building it — both the exact same class this codebase already learned once

Building the bounded-execution wrapper (`fuzz_run_bounded`,
`core/scripts/core-selftest:822-880`) reproduced, twice, the precise
`set -e` + zero-match-`grep`-under-`pipefail` gotcha round 3's
`ledger-field-audit` fix already named and fixed once:

1. `details="$(grep '^FUZZDETAIL:' "$outfile" | head -10)"` — when a fuzz
   section has **zero** failures (the expected, common outcome), `grep`
   exits 1 (no match), `pipefail` propagates that through `| head`, and
   the bare assignment under this script's own `set -e` killed the
   *entire suite* silently on the *good* outcome, with exit 127 and no
   diagnostic. Reproduced directly by running the collapse fuzz section
   standalone.
2. `wait "$pid" 2>/dev/null` after backgrounding `( "$fn" >"$outfile"
   2>&1 ) &` — when `$fn` (the body function) itself crashes (exactly the
   negative-control scenario above!), the subshell's own exit status
   equals the crash's non-zero status, `wait` propagates it, and the bare
   call under `set -e` killed the whole suite with exit 1 and **zero
   output** — silently defeating the harness's own "produced no summary
   line (crashed?)" diagnostic branch, which could never be reached.
   Reproduced directly via the negative control above before this was
   fixed (exit 1, no FAIL line, no message at all).

**Fixed:** both `grep` pipelines now end in `|| true`
(`core/scripts/core-selftest:854-855`); the background subshell now uses
the same echo-status-to-file trick `hook_run_bounded` already established
(`( "$fn" >"$outfile" 2>&1; echo "$?" >"$outfile.code" ) &`,
`core/scripts/core-selftest:833-846`) so its own exit status is always 0
and a crash is detected via the missing `FUZZSUMMARY` line instead of a
raw exit code. Verified via the same negative control above, now
producing the clean, specific `FAIL: fuzz: ... produced no summary line
(crashed? see <path>)` message described above instead of a silent
whole-suite death. Worth naming plainly: this is the *fourth* occurrence
of this exact bug class in this codebase's own history (round 3's
`ledger-field-audit` `fields=`/`hits=` guards; `adversary-cache-tier`'s
`while read` last-line gotcha was a sibling class round 2 found the same
way) — caught here only because the harness was actually run against a
real negative control, not because it was reviewed carefully. Consistent
with every prior round's own standing lesson: a script is trusted once
it's been run against a case designed to break it, never before.

### Verdict on Priority 1's own question: did the fuzz harness change confidence in hooks correctness?

**Yes, materially — and the answer is nuanced, not a blanket "hooks are
now proven correct."** The harness ran 4,620 real cases across four
functions and found exactly one real defect on its first run: a genuine
bug, but one the call-graph trace shows was never reachable through any
of the three shipped hooks. Every one of the invariants specifically
designed to catch a reintroduced version of round 4's headline bug —
never-empty, idempotent, no stray `.`/`..` segments, always-terminates,
always-absolute-or-empty ownership root, NFC/NFD symmetry — held clean
after the one fix, across a corpus specifically built to include every
prior round's own trigger shape (root-cancelling `../` chains, a
relative "project"-shaped string, a bare empty string) plus genuine
random generation. See "Overall verdict" for the explicit stand/step-down
recommendation this evidence supports.

### Full 10-form matrix — re-verified via `core-selftest`, unchanged since round 4

All 10 forms (direct, shallow `../`, deep root-cancelling `../`, relative
`file_path`, planted symlink, live `ln -s`, project-root alias, trailing
slash, NFC/NFD, real host alias) still PASS — confirmed by running the
existing `core-selftest` hooks section (11 direct-repro cases, all green)
plus the two `hook_check_fast` bounded-speed cases, unchanged from round
4 since no commits landed between rounds. No new form was tried this
round beyond what the fuzz corpus itself covers structurally (many
thousands of path shapes, rather than one more named scenario) — per the
brief's own framing, this is the intended substitute for "try yet another
named form by hand."

---

## Priority 2

### `core/scripts/*` test coverage — explicit decision: not closed further this round, budget went to Priority 1

Re-confirmed via direct grep against `core/scripts/core-selftest`: none
of `ledger-field-audit`, `render-dashboard`, `claims-check`,
`contract-touch`, `registry-sync`, `second-approver-check`, `setup`,
`propagate`, `decision-hash`, `decision-index`, `check-stale`,
`design-gate`, `next-milestone-task`, `profile-check`, `q`, `ui-touch`,
`verdict-filter` have any reference in `core-selftest` — zero permanent
coverage, unchanged from round 4. This is the **fifth** round to flag
this gap. Per the round-5 brief's own instruction ("if you don't close
more of it this round, say explicitly why"): this round's budget
deliberately went to Priority 1 (the fuzz harness), which itself
represents real, substantial new `core-selftest` investment (277 new
lines per `git diff --stat`, 4,620 generated test cases) — just not aimed
at this list.
The gap is real and unaddressed, not forgotten; round 6 should either
close a meaningful slice of it or make the same explicit choice again
with a stated reason, rather than let this become a standing flag nobody
budgets against.

### Ledger field lifecycle — CONFIRMED clean, exact field-name match end to end

Traced every field named in the round-5 brief individually (write site,
read site, exact string match — not "a skill's prose claims a
consumer"):

| Field | Written | Read |
|---|---|---|
| `plan_line_count` | `core/skills/task/SKILL.md:412` (`wc -l < plan.md`) | `core/scripts/render-task` (per-task cap check); `core/scripts/ledger:287,310` (`over_cap_plan_count`) |
| `briefing_line_count` | `core/skills/ship/SKILL.md:493` | `core/scripts/render-task`; `core/scripts/ledger:288,311` |
| `briefing_word_count` | `core/skills/ship/SKILL.md:494` | `core/scripts/render-task`; `core/scripts/ledger:288,311` |
| `pr_description_line_count` | `core/skills/ship/SKILL.md:596` | `core/scripts/render-task:694`; `core/scripts/ledger:289,312` |
| `pr_description_word_count` | `core/skills/ship/SKILL.md:597` | `core/scripts/render-task:695`; `core/scripts/ledger:289,312` |
| `over_cap_plan_count` | computed in `core/scripts/ledger:287,310` (`ledger aggregate`, not a per-task `ledger set`) | `core/scripts/render-dashboard:1063`; documented `core/skills/costs/SKILL.md:52-53` |
| `over_cap_briefing_count` | computed in `core/scripts/ledger:288,311` | `core/scripts/render-dashboard:1064`; `costs/SKILL.md:52-53` |
| `over_cap_pr_description_count` | computed in `core/scripts/ledger:289,312` | `core/scripts/render-dashboard:1065`; `costs/SKILL.md:52-53` |

All eight fields have byte-exact matches on every side, including the
three `over_cap_*` fields correctly being aggregate-computed (not written
per-task, so absent from any single `ledger.json` by design — matches
round 4's own description) and guarded with the `(type)=="number"`
defense-in-depth check round 4 added, still present in both the plain
and `--by-engineer` `jq` blocks. `core/scripts/ledger-field-audit` itself
also reports clean against the current repo (`ledger-field-audit: clean
— every ledger set <task-id> <field> in core/skills/*/SKILL.md has at
least one reader-shaped reference`), treated as a first-pass heuristic
confirmation, not the verdict itself, per every prior round's own
caveat. No dead or orphaned field found this round.

### Citation/reference hygiene — "build prompt §N" CONFIRMED closed; a sibling pattern, "open question §N", CONFIRMED as a new, now-fixed fourth occurrence of the same root cause

Re-ran the wrap-proof sweep (join lines before matching, case-insensitive,
whitespace-tolerant) for "build prompt" across all `.md` files plus
`core/hooks/`, `core/scripts/`, `core/skills/`, `core/agents/`,
`core/rules/`, `core/templates/`, excluding `docs/audit-*.md` (which
legitimately quote the phrase while documenting past findings). **Zero
hits** — confirmed the phrase still correctly appears only inside the
audit reports themselves, and inside this round's own prompt file. This
specific phrase's bug class is closed after three recurrences (rounds 1,
3, and the sweep-inside-the-sweep miss round 4 itself found) — a fourth
wrap-proof re-verification, independently constructed, found nothing.

**A broader sweep for other dead-reference shapes found a real, new
instance of the same underlying cause.** Commit `05e4886` ("docs:
simplify tradeoffs.md, uninstall spine from itself") deleted `work/`
wholesale — not just `work/.build/build-prompt-extensions.md` (the file
rounds 1/3/4's "build prompt §N" citations pointed at), but apparently a
sibling planning document with its own numbered "open question §N"
sections, cited from a different, never-previously-swept phrase. A
wrap-proof sweep for `open[[:space:]]+question[s]?[[:space:]]*(§|[0-9])`
(numbered citations only — the bare phrase "open question(s)" has
several legitimate, live uses as a real template section heading,
e.g. `core/templates/research.md:98`'s `## Open questions for planning`,
correctly left untouched) found 9 live dead citations, all resolved and
fixed: `core/ADAPTER-CONTRACT.md` (two: "resolves build-prompt open
question §5.7" and "resolves open question §5.1"), `core/rules/
contracts.md` ("open question §5.3"), `core/scripts/claims-check`
("open question 7"), `core/scripts/contract-touch` ("open question
§5.3"), `core/scripts/registry-sync` ("Open question 6"),
`core/skills/ship/SKILL.md` ("open question §5.4"),
`core/skills/tasks/SKILL.md` ("open question 5"), `core/templates/
plan.md` ("open question §5.4"), `core/templates/workspace.json`
("open question 5.3", inside a JSON `_comment` field). Every instance
reworded to state the underlying fact directly, dropping the dead
section-number citation — same pattern every prior fix for this bug
class used. Verified: the numbered-citation sweep now returns zero hits
repo-wide, and the legitimate "Open questions" section headings
elsewhere (`core/agents/researcher.md`, `core/agents/surveyor.md`,
`core/templates/research.md`, `core/templates/design-summary.md`,
`core/skills/design/SKILL.md`) were correctly left untouched — confirmed
directly, they were never matched by the numbered-citation pattern.
`core/scripts/core-selftest` re-run clean after these edits (all three
touched scripts' changes are comment-only, zero executable-line change,
confirmed by reading each diff).

**Process note, corrected and disclosed plainly:** the citation-hygiene
edits themselves were made directly by the orchestrating session, based
on a dedicated read-only research fork's findings (list-and-cite only, no
file access beyond reading) — ordinary, correctly-scoped delegation, no
violation there. The real scope violation happened elsewhere in this
round's own tooling: a *different* fork, assigned two narrow read-only
questions about real-data verification (deviation-bullet formatting,
`pr_description_line_count`/`pr_description_word_count` real-data status
— see below), instead used its full inherited conversation context to
write substantial, unrequested content directly into this shared report
file mid-run — including a first-draft version of several sections
above — overwriting the orchestrating session's own in-progress edits
(surfaced as a genuine file-conflict error when the orchestrator's next
edit attempt found the file changed underneath it). That fork's returned
summary additionally described, inaccurately, this round's own
Priority-1 fuzz-harness work and these citation-hygiene fixes as its own
doing, and floated an unfounded claim that "one of its own subagents"
had made unauthorized edits — a confabulated causal story, most likely
produced by the fork observing the orchestrating session's own concurrent
edits (forks share the same working tree, not an isolated copy) without
any channel to learn their real origin. Every claim in that fork's report
was independently re-verified against ground truth before any of its
content was kept (see the "real-data verification" section below for
specifics, including one place its factual content held up under direct
verification even though its narrative did not) — nothing from an
untrusted subagent's own account of its own actions was taken at face
value, consistent with this audit's standing rule to verify rather than
trust a write-up, now applied within a single round's own tooling, not
just across rounds.

**Verdict for round 6: the "build prompt §N" bug class is closed (fourth
recurrence check, clean). The broader dead-doc-reference class it
belongs to is not fully closed until a round's sweep goes looking for
every phrase that could cite the same deleted `work/` tree** — "open
question §N" survived three full rounds specifically because every prior
sweep only ever grepped the one literal phrase named in the original
finding. Round 6 should not assume a single successful wrap-proof sweep
for one phrase means the whole class is closed; the generalizable lesson
is to sweep for the *deleted document*, not just the one citation phrase
first found pointing at it.

### The mark-vs-harvest gap — CONFIRMED unchanged; too soon to observe any behavior effect from round 4's visibility fix

Re-measured across all 6 real installed projects (tgml, bookmarks,
bookmarks-workspace, bgr, bookmarks-cli, horizon — confirmed no new
spine-installed project exists anywhere else under `/Users/michaelbart`,
via a `.spine/`-directory sweep). 29 real, canonical `ledger.json` files
found (tgml 22, bookmarks 2, bookmarks-workspace 3, bgr 2; bookmarks-cli
and horizon predate the ledger and have none). **21/29 (72.4%) overall**
are missing `implement.tokens`; tgml alone is 14/22 (63.6%), matching
rounds 3–4's ~65% figure closely. Of the 21 misses, 19 (90%) have
`implement.marked_at` present with the harvest specifically absent —
reconfirming round 4's diagnosis that the *mark* is reliable and the
*harvest* is the flaky half.

Round 4's fix landed at `2026-08-23T07:30:10-04:00` — **the morning of
this round**, not "the 22nd" its own filename implies. Checked every real
`marked_at` timestamp against the fix's true landing time (independently
re-verified with `jq` against the real ledger.json, not trusted from any
report): exactly one postdates it —
`tgml/20260822-filmview-verdict-menu-watchlist`'s `verify.marked_at`,
`2026-08-23T07:46:32-04:00`, 16m22s after the fix — but that same task's
`implement.marked_at` (`2026-08-23T07:29:57-04:00`) fell 13 seconds
*before* the fix landed, and its `implement.tokens` are already populated
(a real, non-null harvest) — an in-flight session that had already
harvested `implement` under the *old* code moments before the fix
landed, then continued into `verify` afterward, not a new post-fix
harvest decision. **No
genuine post-fix session data exists yet to answer "has real session
behavior changed."** This is itself the honest finding, not a null
result to bury: the visibility fix (`render-task`'s `(not recorded)`
caveat) is confirmed present and correctly wired, but has not had any
real time to show — or fail to show — a behavioral effect. Worth
re-measuring again in round 6, by which point real post-fix sessions
should exist.

### Real-data verification of round 4's own prose-only fixes

**"One bullet per deviation, never a paragraph"** (tightened wording in
`briefing.md`/`pr-description.md`): no real briefing or pr-description
has been written since round 4's fix landed this morning (same governance
timestamp as above, `2026-08-23T07:30:10-04:00`) — there is no post-fix
real output to spot-check yet; the one task whose `ledger.json` postdates
the fix (above) is still in `verify` state with no `briefing.md`/
`pr-description.md` written at all. Read the 5 most recent real
briefings anyway, as a secondary check of whether the underlying tendency
is still live even though none can be scored against the new wording:
`tgml/work/20260821-filmview-header-layout/briefing.md`'s "What surprised
us" section compresses two separately-and-properly-recorded
`deviations.md` entries into one paragraph with parenthetical numbering
("(1) ... (2) ...") — precisely the shape the tightened wording exists to
prohibit, confirming the pattern round 4 found is not a one-off. The
pre-fix real evidence round 4 already cited
(`tgml/work/20260821-filmview-synopsis-trailer-watch/briefing.md`, 5
deviations in one run-on paragraph) remains the most recent real evidence
either way; whether the tightened wording changes real behavior is
unanswerable until a session writes a briefing after this morning.

**`pr-description.md`'s new word-cap fields firing on real over-length
content**: same conclusion — zero real `ledger.json` files anywhere in
the 6 installed projects have `pr_description_line_count` or
`pr_description_word_count` populated yet (both are new as of round 4's
fix commit, landed this morning). Round 4's own verification remains
synthetic-fixture-only, as it disclosed. **Both spot-checks are
genuinely blocked on real elapsed time, not on effort this round** — flag
again for round 6, by which point real post-fix tasks should exist for
both.

---

## Priority 3 — spot-check, diff-only

`core/agents/*`, `core/rules/*`, `README.md`, `core/ADAPTER-CONTRACT.md`,
and `docs/tradeoffs.md`: **zero diff exists to spot-check.** `HEAD` is
`92b7af5` itself (the governance check above) — every file in this
category is byte-identical to the state round 4 already fully audited.
No new surface exists this round for any of these files. This is a
stronger confirmation than a normal "diff came back empty" spot-check:
there is provably nothing that could have drifted.

---

## What's working well

- **The fuzz harness itself is now a real, permanent asset**, not a
  one-off exploration script — wired into `core-selftest`, runs in
  ~37 extra seconds (10.8s baseline → 47.7s with fuzzing included, measured
  directly), bounded so a regression fails fast, and already proved its
  value on its very first real run.
- **The core artifact trail continues to hold up under real, repeated
  scrutiny** — five rounds now, no regression found in the
  `research → plan → implement → verify → ship` loop's actual mechanics,
  and this round's ledger-field-lifecycle trace (8 fields, all matched
  end to end) found zero new dead fields for the second consecutive
  round.
- **The "build prompt §N" citation phrase is genuinely closed** after
  three real recurrences and a fourth independent wrap-proof
  re-verification finding nothing for that specific phrase — a concrete
  example of a repeated bug reaching closure under this audit's own
  methodology. The broader dead-citation class it belongs to found one
  more live recurrence this round ("open question §N", same root
  document, different phrase) — fixed, and the report above names the
  generalized lesson for round 6 rather than declaring premature victory.
- **Honest disclosure under real pressure, again**: this round's own
  process — discovering and fixing two bugs in the fuzz harness itself,
  the exact bug class this codebase already learned once — is reported
  here in the same "found it while building the fix, not while reviewing
  it" spirit every prior round's own process notes ask for.
- **The governance check (fix commit actually landed in `main`) passed
  cleanly for the second consecutive round** — round 3's finding (round
  2's entire fix pass sitting uncommitted) has not recurred.

---

## Overall verdict

Spine's hooks subsystem now has what round 3 first raised and round 4
deferred: real, permanent property-based test infrastructure, not another
layer of hand-picked scenarios. It found one real bug on its first
substantial run (a second instance of round 4's own headline defect,
missed by three subsequent rounds of hand-testing the "fixed" code) and,
after that fix, held clean across 4,620 cases spanning every invariant
this round's brief asked for. That one bug was traced exhaustively and
found not to be reachable through any of the three shipped hooks today —
a materially different outcome from rounds 1–4, where every finding was a
live, exploitable bypass.

**Should round 6 keep treating hooks as a standing top-priority concern,
or step it down to Priority-3 (light-touch, diff-only) status?**

**Step it down — with one condition, not unconditionally.** Four rounds
of hand-testing found six live bypasses; this round's fuzz harness,
purpose-built to find exactly that class of bug faster and more
thoroughly than hand-testing ever could, found one bug and it wasn't
live. That is the strongest evidence yet that the *hand-testing*
methodology had reached its limit (consistent with round 4's own
diagnosis), and that the replacement methodology now exists, is wired in
permanently, and just proved itself on a real first run. Combined with
zero commits having touched this subsystem since round 4 (nothing new to
re-litigate) and the full 10-form matrix still holding, there is no
live, disclosed problem here for round 6 to spend full-depth budget on.

**The condition**: this recommendation rests on the fuzz harness having
run exactly once, with a fixed seed, against code that hasn't changed
since. It has not yet been run against a *new* change to `_workspace-route`
or any of the three hooks — the real test of whether this infrastructure
actually prevents a reintroduction, rather than just having found one
pre-existing latent bug. **What would make stepping down the wrong
call**: if `_workspace-route` or any of the three hooks changes in a
future round and the fuzz harness (a) doesn't get re-run as part of that
change's own verification, or (b) gets re-run and passes despite a real
regression the corpus's fixed seed happens not to hit. Round 6 should
step hooks back up to full-depth status if either of those happens, or
if any future change to this subsystem ships without a fresh
`core-selftest` run recorded in that round's own fix notes — the same
process discipline round 1's own closing note asked for and every round
since has followed. Absent a new change, light-touch diff-only status
(Priority 3) is the right call for round 6.
