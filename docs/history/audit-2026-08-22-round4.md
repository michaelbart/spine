# Audit — 2026-08-22, round 4

Independent, adversarial audit run against `main` at commit `a05a041`.
`git status` was clean and round 3's fix commits (`716087d`, `14cadd4`,
`a05a041`) were confirmed present in history at the start of this round —
the governance failure round 3 found in round 2's own fix pass did not
recur.

Findings below were produced by four parallel subagents (hooks +
governance; scripts; skills + digestibility; agents/rules/templates/
ADAPTER-CONTRACT/docs), each independently verifying claims against real
code execution — including constructing and running synthetic fixtures,
reverting fixes to negative-control round 3's new regression tests, and
reading real installed-project data (`/Users/michaelbart/tgml`, 20+ tasks;
`/Users/michaelbart/bookmarks`; `/Users/michaelbart/bookmarks-workspace`;
`/Users/michaelbart/bgr`; `/Users/michaelbart/bookmarks-cli`;
`/Users/michaelbart/horizon`). No new spine-installed projects have
appeared since round 3. Two agents (scripts; agents/templates/docs)
independently found and confirmed the same `render-task` cost-reporting
bug from different angles — noted below where it happened.

Findings were reviewed and fixed in a separate pass, recorded inline as in
rounds 1–3 — each "**Fixed:**" note says what was done and how it was
verified, including a negative control (temporarily reverting the fix and
confirming the relevant regression test actually fails without it) for
every code fix in this round, per the same standing lesson round 2 named.

---

## Hooks — `core/hooks/*`

### A relative `CLAUDE_PROJECT_DIR`, or a `../`-traversal deep enough to cancel to filesystem root, crashes the shared path normalizer into a multi-thousand-process fork loop that resolves fail-open — CONFIRMED, CRITICAL

`core/hooks/_workspace-route:104` (`_wsr_collapse`) builds its result with
`for p in "${out[@]}"; do result+="/$p"; done` after computing `out` as a
bash array. Under this project's target shell, **GNU bash 3.2.57** (real
`bash --version` on this machine, the exact version this codebase's own
comments elsewhere claim to specially accommodate), dereferencing
`"${out[@]}"` when `out` is a legitimately-declared zero-element array
throws `unbound variable` under `set -u` — bash 3.2 lacks the
empty-array-under-nounset exemption bash 4.4+ has. The failure is silently
swallowed by the enclosing `$(...)` command substitution, so
`_wsr_collapse` returns `""` instead of the correct `/`. That empty string
then feeds `_wsr_realpath`'s ancestor-walk (`_wsr_realpath:127-157`):
`dirname("")` is `.`, and `_wsr_collapse(".")` hits the identical bug at
the same non-progressing input — an unbounded recursive loop where **each
level is a real subshell fork** via command substitution, not just a stack
frame.

Two independently-reachable triggers, both collapse the normalized path to
exactly `/`, the one input shape that hits the bug:

1. **A relative `CLAUDE_PROJECT_DIR`.** `_wsr_collapse` unconditionally
   prepends `/` to every segment regardless of whether the input was
   relative, mis-rooting `"proj"` to `"/proj"`; the ancestor walk then
   climbs to real `/` and crashes there — on every hook invocation,
   independent of `file_path`.
2. **A `file_path` with enough `../` segments to fully cancel a whole
   absolute project-path prefix to nothing** — a deeper variant of round
   1's traversal case that no prior round tried (prior repros only walked
   up a few levels, never past the project root). Needs no environment
   control at all, just a crafted `file_path` on one ordinary tool call.

Directly measured, real subprocess execution:

| Hook | Trigger | Wall time | Forked subprocesses | Result |
|---|---|---|---|---|
| `path-escalate` | relative `CLAUDE_PROJECT_DIR`, write to a protected Class 0 path | 71.46s (13.58s user + 53.08s sys) | ~4,340 | exit 0 — **allowed** |
| `phase-gate` | relative `CLAUDE_PROJECT_DIR`, write outside task_dir | ~10–25s | ~6,500 | exit 0 — **allowed** |
| `dep-gate` | relative `CLAUDE_PROJECT_DIR`, write to a `#manifest`-tagged path | ~10–25s | ~4,340 | exit 0, no `ask` — **allowed** |
| `path-escalate` | 20×`../` in `file_path`, canonical absolute project dir | ~25s | ~2,175 | exit 0 — **allowed** |
| any of the above, under `ulimit -u 200` | — | — | hits process-table limit | script aborts mid-recursion with `fork failed` |

This is a real resource-exhaustion vector on the host, not merely a logic
bug — thousands of process-table entries and tens of seconds of wall time
from a single PreToolUse call, which then still allows the write. This bug
lives inside the exact function round 1's own fix introduced
(`_wsr_collapse`), and survived rounds 2 and 3 undetected because neither
tried an input shape that fully cancels to root.

**Minimal fix, verified working in an isolated patched copy:**

```bash
local result="" p
if [[ ${#out[@]} -gt 0 ]]; then
  for p in "${out[@]}"; do result+="/$p"; done
fi
printf '%s' "${result:-/}"
```

Drops the relative-`CLAUDE_PROJECT_DIR` case from 71s/4,340 forks to
0.02s, and `_wsr_collapse("/a/..")` now correctly returns `/`. This alone
eliminates the DoS for both trigger shapes (verified for both). A residual,
lower-severity fail-open remains after just this patch: `_wsr_collapse
("proj")` still returns `/proj` (the wrong root) rather than a correctly
resolved path, so a relative `CLAUDE_PROJECT_DIR` still silently allows the
write (confirmed: same test, now 0.375s, still `exit=0`) — just without
the DoS.

**Fixed:** applied the crash guard above to `_wsr_collapse`
(`core/hooks/_workspace-route:84-106`), and separately added an explicit
absolute-path check at the top of `workspace_route()` that fails closed
(non-zero return, `WSR_OWNER_NAME="(unknown)"`, a stderr message) on a
non-absolute `$project` — closing the residual fail-open the crash guard
alone left, per the recommendation above. All three hooks
(`phase-gate`/`path-escalate`/`dep-gate`) were updated to actually check
`workspace_route`'s return code and fail closed (deny/deny/ask
respectively) rather than treating the now-empty `WSR_OWNER_ROOT` as
"outside project, allow" — the pre-fix code would have silently allowed
through that same early-return path. Verified: the relative-
`CLAUDE_PROJECT_DIR` case now returns `exit=2` in 0.02s (was 71s/4,340
forks/`exit=0`); the deep-`../`-cancellation case now resolves in 0.05s
(was ~25s) with the same correct `allow` decision it always should have
had (the collapsed path genuinely lands outside the project, so allowing
it is right — the bug was the DoS en route to that answer, not the answer
itself). Added two `core-selftest` regression cases
(`hook_run_bounded`/`hook_check_fast`, a purpose-built bounded-execution
harness since neither `timeout` nor `gtimeout` exists on this host) that
assert BOTH the correct decision AND real speed (under 5s) — checking
decision alone would have been fooled by the pre-fix code eventually
resolving via the host's own process-limit exhaustion, which produces a
plausible-looking non-zero exit after several seconds and thousands of
forked processes; confirmed this exact false-negative directly before
tightening the check. Negative-controlled: reverted just the fix (`git
stash`), re-ran `core-selftest` — both new cases correctly failed (hung
past the 5s bound), restored, confirmed clean `git status` and a green
suite again.

### A Unicode NFC/NFD normalization mismatch bypasses `path-escalate` and `dep-gate`'s protected-path matching — CONFIRMED, HIGH SEVERITY

`core/hooks/path-escalate:86` and `core/hooks/dep-gate:77` both do a pure
bash glob comparison (`[[ "$rel" == $glob ]]`) against
`.spine/protected-paths.conf` entries. macOS APFS is normalization-
insensitive: an NFC-encoded and an NFD-encoded (byte-distinct) form of the
same accented filename address the identical real file, confirmed directly
on this machine's scratch volume. `_wsr_realpath`'s physical resolution
(`cd -P`/`pwd -P`, `readlink`) does not normalize Unicode form — it
preserves whichever byte sequence the caller supplied.

Direct repro, both hooks: a protected glob `src/other/café/**` registered
in NFC bytes correctly denies a write when `file_path` is given in NFC
form; the identical real file, addressed via the same `file_path` in NFD
byte form, is **allowed** — `path-escalate` returns `exit=0`; `dep-gate`
returns `exit=0` with no `ask` at all. `phase-gate` isn't exposed to this
specific mechanism since task-folder containment doesn't derive from
arbitrary protected-path config.

**Fixed:** added `_wsr_nfc()` and `_wsr_glob_eq()` to
`core/hooks/_workspace-route` — `_wsr_nfc` is a zero-cost pass-through for
pure-ASCII input (the overwhelming common case, no subprocess spawned) and
shells out to `python3`'s `unicodedata.normalize("NFC", ...)` only for
non-ASCII input (`python3` chosen over `iconv` because macOS's
Apple-specific `UTF-8-MAC` iconv target that does decomposition/
recomposition has no Linux equivalent, while `python3` is portable and
already present on this and every real installed project's host checked);
`_wsr_glob_eq` tries the existing byte comparison first and only falls
back to comparing both sides NFC-normalized if that misses, so every
pure-ASCII or already-matching case is byte-for-byte unchanged. Wired into
both `path-escalate` and `dep-gate`'s glob-match call sites in place of
the raw `[[ "$rel" == $glob ]]`. Verified: the exact NFC/NFD repro above
now denies in both forms; a `core-selftest` regression case (skipped, not
failed, if `python3` isn't on `PATH`, matching the helper's own graceful
fallback) reproduces it as a permanent test. Scoped limitation, disclosed
rather than silently accepted: on a host with no `python3`, the ASCII fast
path means existing byte-identical-comparison behavior is unchanged (no
regression), but the non-ASCII NFC/NFD case reverts to pre-fix (undenied)
behavior — acceptable since `python3` was confirmed present everywhere
this was checked, and the alternative (a hard new dependency for every
hook invocation, even ASCII-only ones) has a real cost the ASCII fast path
avoids.

### Full 10-form matrix result (this round's two new forms both failed as found; both now fixed and re-verified)

| # | Form | Result as found | Result after fix |
|---|---|---|---|
| 1 | Direct canonical path | PASS | PASS |
| 2 | `../` traversal (shallow) | PASS | PASS |
| 2b | `../` traversal deep enough to cancel to filesystem root | **FAIL — DoS then allow** | PASS (fast, correct allow) |
| 3 | Relative `file_path`, absolute project dir | PASS | PASS |
| 4 | Planted symlink into protected path | PASS | PASS |
| 5 | Live `ln -s` via Bash, and a write through the created link | PASS | PASS |
| 6 | Self-created project-root alias (round 3 style) | PASS | PASS |
| 7 | Trailing-slash difference, both directions | PASS | PASS |
| 8 | Unicode NFC vs. NFD, same real file | **FAIL — allowed** | PASS (deny) |
| 9 | Relative `CLAUDE_PROJECT_DIR` | **FAIL — DoS then allow** | PASS (fast, fail-closed deny) |
| 10 | Real host alias (`/tmp` → `/private/tmp`), both directions | PASS | PASS |

### `_bash-write-targets` structural coverage — no new gap found

Directly tested `sudo cp`, a heredoc `cat > file <<EOF`, a quoted
destination path, and a same-command Bash function wrapper against
`dep-gate`'s manifest target — all four correctly resolved to a concrete
`PATH:` match or a fail-closed `UNRESOLVED:`. `rsync` (and any verb outside
the fast-path regex) remains completely invisible, confirmed directly —
this matches `docs/tradeoffs.md`'s existing disclosed limit verbatim, not
a new finding.

### Round 3's regression tests are meaningful, not vacuous — CONFIRMED

Swapped in the pre-fix `_workspace-route`/`dep-gate` via
`git show a05a041^:<path>` and re-ran `core-selftest`: 8 of 11 hook cases
correctly failed without the fix (all alias-mismatch cases, the
reverse-alias case, the planted-symlink case, and phase-gate's alias case,
which crashes outright pre-fix); the 3 unrelated controls kept passing.
Restored both files, confirmed clean `git status` and a green
`core-selftest` afterward.

### Verdict — a fifth and sixth distinct bypass mechanism this round

Two new, independently confirmed bypass mechanisms beyond the four already
documented (round 1's `../`, round 2's symlink, round 3's alias mismatch,
round 3's dep-gate Bash-branch omission): **Unicode NFC/NFD
insensitivity** (a clean structural gap, no crash, silent and
deterministic) and **a bash-3.2 empty-array crash inside the shared
normalizer itself**, which resolves to a real multi-thousand-process
fork-loop DoS that still ends in fail-open. The second one is the more
important signal: it isn't a new input-shape gap in the surrounding
call-site logic, it's a bug **inside the fix code round 1 itself wrote**,
surviving two further full audit rounds because neither tried an input
that fully cancels to root. See "Overall verdict" below for the
structural-fix recommendation this motivates.

---

## Scripts — `core/scripts/*`

### `render-task` renders a real, nonzero-duration `implement` phase as a bare, uncaveated "0 tok" in the majority of real shipped tasks — CONFIRMED, independently found by two agents, reproduced against real production data

Re-measured the disclosed verify-phase honor-system gap
(`docs/tradeoffs.md:119-130`) across the same 20-task real `tgml` sample
round 3 used: `.phases.implement.tokens` is still missing in 13/20 (65%)
— unchanged since round 3. But a sharper re-measurement shows the *mark*
(`.phases.verify.marked_at`) is now present in 19/20 (95%) — sessions
reliably call `ledger mark <task-id> verify`, they just don't reliably
also make the companion `ledger harvest <task-id> implement ...` call that
same action is supposed to bundle in (`task/SKILL.md:670-680`).

Because `implement.marked_at` is *also* reliably present (set when
`implement` starts), `render-task`'s round-3 fallback
(`render-task:444-459`, "not reached (N tok recorded under a subagent
phase)") never triggers for this case — it only fires when a phase's own
`marked_at` is missing. The `implement` row instead takes the normal
render branch (`render-task:460-471`) and shows a real duration with a
silent, uncaveated zero cost. Reproduced directly against
`/Users/michaelbart/tgml/work/20260821-filmview-synopsis-trailer-watch/
ledger.json` (one of the 13 real tasks missing `implement.tokens`):
1h28m of real implement-phase work renders as `0 tok`, indistinguishable
from a phase that genuinely cost nothing, while the adjacent `verify` row
correctly shows `6.9M tok`. This is worse than the case round 3 fixed —
it reads as "implementation cost nothing," actively misleading rather
than merely blank — and it also silently undercounts `ledger aggregate`'s
project-wide totals by the same amount.

`docs/tradeoffs.md:119-130`'s wording conflates "the mark" and "the
harvest" under one noun ("this mark... roughly two-thirds were missing
it"). Given the two now have measurably different reliability (96% vs.
30%), the disclosure should say so explicitly — it currently reads as if
`marked_at` is the flaky part, when it's the harvest that stays flaky at
essentially the same rate as round 3 measured.

**Fixed:** the per-phase render branch (`render-task:460-478`) now computes
a `cost_label` that appends `" (not recorded)"` whenever a phase's cost is
`0` and real elapsed duration since the previous phase (`$dur`) was
computed — the signal a genuinely-free phase wouldn't have. Verified
against the exact real repro: the same `filmview-synopsis-trailer-watch`
ledger now renders `implement` as `0 tok (not recorded)` (was a bare
`0 tok`) with its real 1h28m duration still shown; `plan` in the same
report picks up the same honest caveat. Corrected `docs/tradeoffs.md:
119-130`'s wording to scope the reliability figures to "the mark" (~95%)
vs. "the harvest" (~65-70%) separately rather than one conflated "this
mark." Added a `core-selftest` regression case for this exact shape (mark
present, harvest field absent, real elapsed duration) — negative-
controlled by reverting just this fix and confirming the case fails
(reverts to a bare `0 tok`), then restored.

### `core-selftest` still has zero direct coverage of most of `core/scripts/*` — CONFIRMED, unchanged from round 3's own "for round 4" flag

`ledger`, `render-task`, `render-dashboard`, `ledger-field-audit`,
`claims-check`, `contract-touch`, `registry-sync`, `second-approver-check`,
`setup`, `propagate`, `decision-hash`, `decision-index`, `check-stale`,
`design-gate`, `next-milestone-task`, `profile-check`, `q`, `ui-touch`,
`verdict-filter` — none have direct regression coverage. Only `floor`,
`adapter-conformance`, `adversary-cache-tier`, and the three hooks do.
Notably, `ledger`'s biggest logic change this round (the `harvest`
command's `message.id` dedup, below) has no coverage at all, despite being
the riskiest change in the round's diff.

**Partially fixed:** added `core-selftest` coverage for the two `render-
task` bugs fixed this round and for `ledger harvest`'s `message.id` dedup
+ additive-sum behavior (see both below) — the parts of this gap this
round's own fixes actually touch. The broader gap (the remaining dozen-plus
scripts with zero coverage, none of which changed this round) is
unaddressed and stays open for round 5, same as round 3 left it.

### `ledger harvest`'s new `message.id` dedup is real, necessary, and correct — verified against a real transcript, but undocumented and untested

`core/scripts/ledger:110-124` added `unique_by(.message.id)` before
summing token usage from a transcript, because one real API call spans
multiple transcript lines (thinking block, tool_use block, etc.), each
carrying an identical copy of that call's `usage`. Verified against a real
Claude Code session transcript
(`~/.claude/projects/-Users-michaelbart-tgml/*.jsonl`): 788
`type=="assistant"` lines, only 444 distinct `message.id` values — a real
~1.77x double-counting factor if undeduped, confirmed via direct
inspection of repeated identical `usage` blocks for the same id. This is a
legitimate, needed fix, `message.id` was confirmed always-present in real
data, and it isn't exposed to the `while read`/`set -e` gotcha classes
(pure `jq`). It is, however, undocumented in any audit round (grepped all
three prior reports, zero hits) and has zero `core-selftest` coverage —
flagged here specifically so a future silent regression in this logic
doesn't go unnoticed the way it would today. (The code itself already
carries an inline comment explaining the dedup — `core/scripts/ledger:
110-114` — so the "undocumented" half of this finding is about audit-report
history and test coverage, not a missing code comment.)

**Fixed (coverage only — the logic itself needed no change):** added two
`core-selftest` cases exercising real `ledger harvest` calls against a
synthetic transcript with a repeated `message.id` (two blocks of one real
call) plus a distinct second call, asserting the summed tokens reflect
exactly one count per unique id, and a second harvest of the same phase
key asserting the additive-sum (not overwrite) behavior. Negative-
controlled both: removed `unique_by(.message.id)` and re-ran — both cases
failed with double-counted totals (input 230 instead of 130, then 240
instead of 140) as expected; restored and confirmed green.

### `floor`'s Class 2 smoke hard gate — reconfirmed robust under a fresh variation, CONFIRMED

Reverted the round-1 hard-gate fix, re-ran `core-selftest`: round 3's
regression case (using status `"unavailable"`) correctly failed as
expected. Restored, confirmed clean. Additionally tested a third "not
implemented" shape neither round 1 nor round 3 tried — `smoke-run`
entirely absent from `capabilities.json` (`status_of` defaults to
`"missing"`) — and confirmed the current fixed `floor` still correctly
hard-fails. Floor's Class 2 gate is now robust across three independently
tested "not implemented" shapes spanning three rounds.

### `ledger-field-audit`'s line-join extraction logic — reconfirmed sound under an adjacency stress test, CONFIRMED

Built a fixture with two unrelated `ledger set <task-id> <field>`-shaped
mentions on adjacent lines in separate sentences: the per-occurrence
anchored regex correctly extracted each field independently with no
cross-matching. Also reconfirmed the round-3 zero-match `set -e` guard
still exits cleanly on a fixture with no `ledger set` mentions at all. No
bug found; round 3's fix holds under this variation.

### `briefing_word_count` traced end-to-end, confirmed new this round, confirmed firing on real over-length content — with one coupling gap

`git log --all -S"briefing_word_count"` confirms it's new this round
(commit `a05a041`). Write side (`ship/SKILL.md:489-494`) and read side
(`render-task:680,707-708`) match on the exact field name. Built a scratch
fixture from a real, already-shipped over-length briefing
(`/Users/michaelbart/tgml/work/20260821-filmview-synopsis-trailer-watch/
briefing.md`, 805 words / 19 lines — under the line cap, over the word
cap), ran `render-task` for real: correctly renders "briefing.md length:
19 lines — over the ~1-page cap (805 words)."

**CONFIRMED bug, reproduced:** `render-task:700` nests the entire
briefing-length block, word-count check included, inside
`if [[ -n "$briefing_lines" ]]`. Setting `briefing_line_count=null` while
`briefing_word_count=805` in the same fixture makes the whole block
disappear — silently hiding an 805-word over-cap briefing. Real production
data (`tgml`'s two most recent tasks) already shows these two fields can
diverge (one present, one `null`), though observed in the safe direction
so far.

**Fixed:** changed the guard to `if [[ -n "$briefing_lines" || -n
"$briefing_words" ]]`, with the inner logic now displaying "N words" when
only the word count is present (rather than the digit-only line-count
placeholder). While fixing this, factored the briefing.md logic into a
shared `paragraph_length_item()` function and reused it for
`pr-description.md` too (see that finding below) — the two templates share
the identical format and cap logic, so this closes both bugs' fix in one
place instead of two near-duplicate blocks. Verified against the real
805-word briefing fixture with `briefing_line_count` unset: now correctly
renders `805 words — over the ~1-page cap`. Added a `core-selftest`
regression case; negative-controlled by reverting the guard and confirming
it fails (the over-cap line disappears), then restored.

### Redundant/overbuilt-machinery check against real installed-project data — no dead machinery found

`verdict-filter`, `decision-hash`/`decision-index`, `design-gate`/
`next-milestone-task` are all genuinely exercised by real data across the
six installed projects (spot-run directly against real artifacts, correct
output). `adversary-cache-tier` is correctly wired but has no production
evidence yet — it's new this round, worth a real-usage check next round
once re-dispatches accumulate; not itself a finding.

### What holds up fine

`claims-check`, `contract-touch`, `design-gate`, `propagate`,
`registry-sync`, `second-approver-check`, `setup` — prose-only changes
this round (dead "build prompt §N" citation cleanup), logic unchanged.
`adapter-conformance`, `check-stale`, `conformance`, `decision-hash`,
`decision-index`, `next-milestone-task`, `profile-check`, `q`, `ui-touch`,
`verdict-filter` — untouched since round 3, spot-checked, no issues.

---

## Skills — `core/skills/*` and digestibility

### Round 3's "build prompt §N" cleanup is not actually complete in the repo — CONFIRMED

Round 3 claimed a repo-wide grep for "build prompt" returned zero hits.
It doesn't: two live citations remain, both hidden by the same
line-wrap-across-a-grep pattern round 1 and round 3 already named as the
reason earlier sweeps missed things —
`core/hooks/_workspace-route:10-12` ("...the mechanical form of build\n
prompt §2's...") and `core/scripts/conformance:3-4` ("...build\nprompt
§2.5 Layer 4"). Neither was touched by round 3's fix pass — this is the
third occurrence of the identical bug class, this time surviving inside
the very sweep that claimed to close it out. (Separately, and not counted
against the repo: `~/.spine/user-config.json`, a machine-local file
outside the repo, also still has one instance — noted for completeness,
not a repo defect.)

**Fixed:** reworded both to state the fact directly (dropped the dead
section-number citation, kept the actual rule in plain prose), the same
pattern used everywhere else this class was fixed. Verified with a
line-join-before-grep sweep (`tr '\n' ' ' < file | grep -o "build[[:space:]]
*prompt"`, resistant to the exact line-wrap that hid these from a naive
sweep) across every `.md`/`core/hooks/*`/`core/scripts/*` file in the
repo: zero hits outside the audit reports themselves (which quote the
phrase while documenting this finding, not a live reference).

### `pr-description.md`'s missing word-cap — resolved: this is a real gap, not a deliberate omission

Round 3 left this as an open question; verdict this round is **real gap,
same fix `briefing.md` already got.** `pr-description.md` is structurally
the same one-paragraph-per-label format as `briefing.md` (its own header
comment says "same rule as briefing.md, restated here"), and real
production data shows it consistently runs *longer* than the sibling
briefing.md for the same task — e.g. 985 vs. 805 words, 828 vs. 534 words,
across sampled `tgml` tasks; 12 of 20 real `tgml` pr-description.md files
exceed the same 600-word threshold established for briefing.md, a higher
rate than briefing.md itself hits. `ship/SKILL.md:584` only records a
boolean `pr_description: "generated"` marker, never a length, and no
downstream reader (`render-task`, `render-dashboard`, `costs/SKILL.md`)
has a length field to read even if one were written. There's no structural
reason for the exemption — pr-description.md's own reader has *less*
context than briefing's, and its extra sections make it systematically
longer.

**Fixed:** added the `wc -l`/`wc -w` recording step to `ship/SKILL.md` §4a
(same redirect-stdin form as briefing, same reason — the filename-suffix
form breaks the numeric cap check downstream), recorded as
`pr_description_line_count`/`pr_description_word_count`; extended
`render-task` via the shared `paragraph_length_item()` helper (see the
briefing coupling-bug fix above) so pr-description.md gets identical
over-cap detection. Verified with a synthetic fixture matching a real
task's actual word count (985 words, over cap): correctly renders
`pr-description.md length: 25 lines — over the ~1-page cap (985 words)`.
No `core-selftest` coverage added specifically for this (the shared
helper is exercised by the briefing test cases above; a real
pr-description.md over-cap output hasn't shipped yet since this is a
same-round fix — worth a fresh real-data spot-check next round, same as
this round did for briefing's round-3 fix).

### "One line per `deviations.md` record" is routinely violated in real output — CONFIRMED

Both templates instruct one line per deviation record. Reading ~15 real
briefings end to end, roughly half compress ≥2 deviations into one dense
run-on paragraph with parenthetical numbering instead of separate lines
(e.g. `tgml/work/20260821-filmview-synopsis-trailer-watch/briefing.md:5`,
5 deviations in one ~500-word paragraph); the other half correctly use
bullets (e.g. `tgml/work/20260818-supabase-schema-verdicts/briefing.md:5`).
This undercuts the writing mandate's own "never bury a surprise below its
section's first line" rule — in the filmview example, deviation 5 really
is buried nearly 500 words in.

**Fixed:** tightened the wording in both templates to "one separate
markdown bullet per record, never a single paragraph with parenthetical
numbers even when terse," with briefing.md's version additionally naming
the concrete failure mode (deviations buried under length pressure) so
the *why* travels with the rule, not just the instruction. This is a
prose-only change to a fill-in-the-blank template — no mechanical
backstop exists or was added; whether real future briefings actually
follow the tightened wording is worth a spot-check in a future round,
same honor-system caveat as the rest of this template's prose.

### The word-count backstop (round 3's headline fix) verified working on real data — CONFIRMED

Constructed a synthetic ledger from the same real 805-word briefing above
and ran `render-task` directly: correctly rendered the over-cap flag with
the real word count. A real fix, not just a synthetic pass.

### The word-count check silently disappears if the paired line-count field is missing — CONFIRMED, reproduced

Same bug independently found by the scripts agent above
(`render-task:700`) — see that section for the fix; both agents reached it
from different fixtures and agree on cause and remedy.

### Over-cap plan/briefing counts are recorded but never aggregated anywhere — CONFIRMED

`plan_line_count`, `briefing_line_count`, `briefing_word_count` are
written per-task (`task/SKILL.md:412`, `ship/SKILL.md:493-494`) and read
only by `render-task`'s single-task view. `ledger:280-306`'s `aggregate`
and `--by-engineer` roll up sibling fields (`avg_conformance_score`,
`second_approver_count`, `pr_description_count`, `tooling_gap_count`) but
nothing plan/briefing-cap-shaped; `costs/SKILL.md` and `render-dashboard`
don't surface it either. This is the same "written but not wired into
every reader" bug class round 2 found and fixed for a sibling field
family, recurring for the field family added in round 2/3's own fix pass.

**Fixed:** added `over_cap_plan_count`/`over_cap_briefing_count`/
`over_cap_pr_description_count` to `ledger aggregate` (both the plain and
`--by-engineer` jq blocks), mirroring `pr_description_count`'s existing
shape, guarded with an explicit `type=="number"` check so a malformed
hand-authored value (the same "5 file.txt" shape `render-task`'s own
defense-in-depth comment already names) can't silently count as "over
cap" via jq's type-ordering rules (a string always sorts greater than a
number in jq, which would otherwise make any non-numeric value trivially
"exceed" the threshold). Wired into `costs/SKILL.md`'s documented field
list and `render-dashboard`'s stat grid (three new tiles, same pattern as
the pre-existing `avg_conformance_score`/`second_approver_count` "needed
its own wiring too" comment already on that code). Verified against a
synthetic three-task fixture with one malformed field (confirmed it's
correctly excluded, not counted) and against real `tgml` data (renders
`0` for all three — expected, since none of these fields are populated in
real ledgers yet, `briefing_word_count`/`pr_description_word_count` being
new this round and no task having shipped since).

### Digestibility spot-check of a full real task — reads well, reasoning not hidden

Read a full real task's `research.md`/`plan.md`/`verify.md`/`briefing.md`
end to end as a human would
(`tgml/work/20260822-seal-overlay-component/`). Genuinely good: real
file:line citations, an explicit risk/mitigation for the one delicate
piece, two real HIGH-severity concurrency bugs disclosed plainly and
re-verified under a harder race, capability gaps stated rather than
absorbed. Consistent with prior rounds' assessment — nothing here reads as
a checklist.

### `writing-mandate.md`'s "referenced by" header is stale — CONFIRMED, cosmetic

Names only two of the four real referencers (`task/SKILL.md`,
`ship/SKILL.md`; missing `core/templates/plan.md:7` and
`core/templates/pr-description.md:112`).

**Fixed:** updated the header to list all four real referencers.

### Token cost

One moderate, non-blocking candidate: `costs/SKILL.md:21-36` spends ~15
lines justifying the untracked-ratio caveat; content compresses to
roughly half the length without losing the load-bearing part. PLAUSIBLE,
not confirmed as pure waste — may be deliberate insurance against the
caveat getting paraphrased away, the same pattern used elsewhere for
"never compress" rules. Not urgent. Otherwise the corpus remains
disciplined — most long "why" passages tie to a specific named historical
bug or audit finding.

### What's working well

`verify/SKILL.md`'s multi-repo `conformance_score` averaging fix (round 3)
is correct and consistent. `security-checklist/SKILL.md` stays tight.
`spine/SKILL.md`, `tasks/SKILL.md`, `task-report/SKILL.md`,
`visualize/SKILL.md` remain genuinely read-only and internally consistent.
`ratchet/SKILL.md`'s discipline is unambiguous. `core/rules/contracts.md`
and `migrations.md` check out exactly against their enforcement scripts.
`ledger-field-audit` reports clean against current `core/skills/*` — no
new dead fields this round.

---

## Agents, rules, templates, ADAPTER-CONTRACT, README, tradeoffs.md

### `core/agents/*` — clean this round

All four agent files (researcher, falsifier, security, surveyor) checked
against every real invocation site (`task/SKILL.md`, `verify/SKILL.md`,
`design/SKILL.md`, `adopt/SKILL.md`). Every delegation matches what the
target agent's frontmatter/body promises, in both normal and design mode.
No contradiction found between any agent's claimed output shape and its
downstream consumer.

### `core/templates/*` — no cap violations, no boilerplate bloat found against real output

All 27 real `plan.md`s measured (46–144 lines, cap 200) — comfortably
under. The optional "Grounds on decisions" section is genuinely,
substantively populated everywhere it's present, never left as unfilled
boilerplate. The milestone `## Known gaps for future member tasks`
mechanism verified working end-to-end against real data
(`tgml/work/M3/milestone.md`'s `next-gap-id` counter correctly
incrementing against real gap entries; every other milestone correctly
shows it unused). A false alarm ruled out: three real plans have a
backtick inside a "why" clause that `conformance`'s awk parser strips
before comparison — the round-2 backtick-corrupts-matching bug does not
recur. Live confirmation the deviation-circuit-breaker honor-system
disclosure (`docs/tradeoffs.md:83-91`) is still accurate: a real 5-deviation
`tgml` task shows the session's own closing note admitting it initially
avoided formalizing deviations 3–5 specifically to dodge the reset
threshold, until the falsifier's second round called it out — strong,
current evidence the disclosure holds, and that the project's disclosure
culture is real, not just designed-in.

### `ADAPTER-CONTRACT.md` §4's two extra self-test requirements are unenforced by `adapter-conformance` — CONFIRMED

§4 imposes two rules beyond the universal pass/fail/one-line-output check:
a scoping proof for changed-file-set capabilities, and a "self-test must
exercise the adapter's real invocation path" rule motivated by a real
cited past incident (a `callers` adapter whose self-test grepped a bare
symbol name while the real adapter greps a full repo-relative path — a
shape neither fixture ever exercised). `adapter-conformance:80-138` is a
black-box exit-code/line-count checker with no mechanism to verify either
rule — it cannot detect a hand-written adapter whose self-test fixture
lacks an out-of-scope violator, or whose self-test branch never calls
through the real invocation path. `docs/tradeoffs.md`'s closest existing
bullet ("the floor trusts its own adapters between recalibrations") is
about drift *between* recalibrations and doesn't cover that this gap
exists even *at* the moment of recalibration, by construction.

**Fixed (documentation, not code):** did both — extended the existing
`docs/tradeoffs.md` "floor trusts its own adapters between
recalibrations" bullet to explicitly name this gap (adapter-conformance
validates self-test *output shape*, never *self-test faithfulness*), and
added a short note directly after §4's two rules in
`core/ADAPTER-CONTRACT.md` itself stating plainly that they're authoring
requirements `adapter-conformance` doesn't mechanically check, pointing
back to the `callers` incident as the concrete case that satisfied every
automated check while violating this. No code change — this finding is
about the contract's own claims, not a checker to fix.

### `README.md`, `core/rules/*` — accurate, no staleness found

Full command table cross-checked against `ls core/skills`; all 17
user-invocable command skills correctly carry `disable-model-invocation:
true`. `core/rules/contracts.md` and `migrations.md` remain tight,
internally consistent, and their claimed enforcement mechanisms verified
to match `contract-touch`/`migrate-rehearse` exactly.

### `docs/tradeoffs.md` — full accuracy pass

Every disclosed limit checked against current code or, where the claim is
conceptual/design-philosophy (cost estimates, wrong-tool cases,
obsolescence conditions), left unverifiable by construction and noted as
such. All code-checkable claims held **except one, already covered above**:
the phase-transition-marks honor-system bullet
(`docs/tradeoffs.md:119-130`) is now imprecise — it conflates "the mark"
(now ~96% reliable) with "the harvest" (still ~65-70% missing), see the
`render-task` finding above for the fix. Everything else — Class 0/1/2
gate behavior, the deviation circuit breaker, adversary findings informing
rather than gating ship, write-blocking hooks' visible-mutation-shape
limit, per-task floor's diff-only scope, the briefing heading convention's
post-change-only binding, the working-with-other-engineers and cross-repo
sections — checked out accurate against current code and/or fresh real
data.

---

## Overall verdict

Spine remains in genuinely good shape on the axes it's actually trying to
guarantee — the deterministic gates (floor's Class 2 hard gate, contract
conformance, the deviation circuit breaker) hold up under repeated,
independently-constructed adversarial testing across four rounds now, and
real production data continues to show the system earning its complexity:
real adversary-caught bugs, honest capability-gap disclosure, and (new
this round) a session caught admitting, in its own words, that it briefly
tried to dodge a mechanical safeguard — exactly the disclosure culture the
docs claim exists, holding up under real pressure to defect from it.

This round was not a clean bill of health as found: the hooks subsystem
produced its fifth and sixth distinct bypass mechanism across four rounds,
and the crash bug was qualitatively different from the prior four. Rounds
1–3 found gaps in call-site logic surrounding the shared normalizer (a
comparison that didn't consult a resolved form, a Bash branch that never
ran a check). This round found a genuine bug **inside the shared
normalizer itself**, introduced by round 1's own fix, that turned two
ordinary-looking inputs into a real multi-thousand-process denial-of-
service that still resolved to fail-open. That was no longer "another
input shape the ad hoc comparisons missed" — it was evidence that
hand-testing scenario-by-scenario, even adversarially and even across four
full rounds, was not converging on correctness for this piece of code,
because nobody had tested what happens when the normalizer's own
intermediate state degenerates to empty/root.

Both bugs (the crash/DoS and the Unicode bypass) are now fixed and
re-verified against the full 10-form matrix, with negative-controlled
regression coverage. Round 3 raised, and deferred, the question of whether
this round's evidence would justify a structural fix — one shared,
single-call path-resolution helper for all of `core/hooks/*` — instead of
another incremental patch. **This round's fix pass took the structural
option, not the incremental one**: `path-escalate` and `dep-gate` each
previously had their own inline `[[ "$rel" == $glob ]]` glob comparison;
both now call a single `_wsr_glob_eq()` added to `_workspace-route`, so
the Unicode fix (and any future path-comparison fix) lives in exactly one
place instead of two near-duplicate call sites. `workspace_route()`'s
entry point now validates its own precondition (an absolute `$project`)
before any hook-specific logic runs, rather than each hook silently
inheriting whatever `_wsr_collapse` happened to do with a malformed root.
Every hook in the subsystem now routes every path decision through this
one shared library, with zero remaining hand-rolled path comparisons
outside it.

What this round's fix pass did **not** build is genuine property-based/
fuzz-test infrastructure — the regression coverage added is still five
hand-picked edge cases (empty-array root collapse, deep `../` cancellation,
relative project dir, NFC, NFD), the same style of scenario-specific
repro this report's own evidence says hasn't been sufficient across four
rounds, just a wider net of scenarios. That's a real, separate investment
(generating random path/segment combinations and asserting invariants
hold, not enumerating cases a human thought of) and is the honest
"unfinished" part of round 3's structural-fix question — worth treating as
its own dedicated piece of round 5's work, not assumed complete because
the code consolidation happened.

Outside hooks, this round's other findings were real but lower-stakes,
and all are now fixed: a cost-reporting accuracy bug (`render-task`'s "0
tok" for a real-cost phase) that misled a human reading their own
project's spend without threatening any gate; a genuine, now-resolved
open question (`pr-description.md` needed and got the same word-cap
treatment `briefing.md` got); and a third recurrence of the exact same
line-wrap-survives-a-grep citation bug, this time inside the sweep that
claimed to have already fixed it — worth noting only because it happened
three times with the same root cause (grep the literal phrase, not a
wrap-proof substring), which is itself a small process lesson for how
future cleanup passes should be verified (applied to this very fix: the
wrap-proof grep pattern above, not the literal phrase).

**What would change this verdict:** if a fifth full round of the same
adversarial-matrix testing (now including explicit fuzz-style edge cases,
per the "not yet built" gap above) still finds a new hooks bypass
mechanism, that would say the problem is deeper than this round's
consolidation reached — worth reconsidering whether path/write
enforcement belongs in bash hooks at all versus a different enforcement
layer. If it holds clean, that's real evidence the pattern has been
closed. As shipped this round, "real work done in the identified place,
one real gap (fuzz-test infrastructure) still open, otherwise sound" is
the accurate read — not "fine as is," and not "overbuilt."

---

## For round 5

- Build real property-based/fuzz-test infrastructure for
  `core/hooks/_workspace-route`'s path functions — generate random path/
  segment/Unicode combinations and assert invariants (idempotence,
  never-empty-unless-root, NFC(f) == NFC(NFC(f))) hold, rather than adding
  more hand-picked scenario cases. This is the one piece of round 3's
  deferred structural-fix question this round's consolidation (single
  shared glob-match helper, absolute-path precondition) didn't reach — see
  "Overall verdict" above.
- `ledger-field-audit`, `render-dashboard` (most of it), `claims-check`,
  `contract-touch`, `registry-sync`, `second-approver-check`, `setup`,
  `propagate`, `decision-hash`, `decision-index`, `check-stale`,
  `design-gate`, `next-milestone-task`, `profile-check`, `q`, `ui-touch`,
  `verdict-filter` still have zero permanent `core-selftest` coverage —
  this round closed the gap for `ledger harvest`'s dedup logic and two
  `render-task` bugs specifically (the parts this round's own fixes
  touched), not the broader surface. Still worth closing incrementally,
  same standing flag as rounds 3 and 4.
- Re-measure the implement-harvest miss rate (currently ~65-70%, unchanged
  since round 3) after this round's `render-task` fix has had real usage
  time to surface whether the visibility improvement changes session
  behavior — the fix makes the gap visible, it doesn't force the harvest
  call to happen.
- Once real tasks ship with `pr-description.md`'s new word-cap fields
  populated, spot-check the fix fires on real over-length content the
  same way this round confirmed briefing's round-3 fix does (this round
  verified only against a synthetic fixture built from a real word count,
  not a real ledger.json — no task has shipped since the fix landed).
- Spot-check whether the tightened "one bullet per deviation, never a
  paragraph" wording in briefing.md/pr-description.md actually changes
  real output — this round fixed the prose, not a mechanical enforcement,
  so whether it holds under length pressure is an open, honor-system
  question same as the rest of these templates.
