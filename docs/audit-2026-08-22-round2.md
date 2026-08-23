# Audit — 2026-08-22, round 2

Independent, adversarial audit run against `main` at commit `14cadd4`
("fix: hook path-traversal bypass, core-selftest regression, dead ledger
fields") — the commit that closed out the same-day `docs/audit-2026-08-22.md`
round. This round assumes that round's fixes are real (spot-checked several
directly: `core-selftest` reproduced green, the `../` traversal fix
reproduced closed on both `path-escalate` and `phase-gate`) and goes looking
for what's still wrong or newly introduced.

Findings below were produced by one orchestrating pass (hooks, direct
testing) plus three parallel subagents (skills; scripts;
agents/templates/docs), each independently verifying claims against real
code execution, and where possible against real installed-project data
(`/Users/michaelbart/tgml`, 29 tasks; `/Users/michaelbart/bookmarks`, 6;
`/Users/michaelbart/bookmarks-workspace`, 4, multi-repo). This report was
produced as a report-only pass (no fixes applied during the audit itself,
per the ground rules governing that pass); every finding below was then
reviewed and fixed in a separate pass, recorded inline as it was in round
1. Each "**Fixed:**" note says what was done and how it was verified —
directly against real code execution, real fixture reproductions, or (for
the hooks) real installed-project-shaped path aliasing, not by re-reading
the diff.

---

## Hooks — `core/hooks/*`

### Symlinks defeat all three enforcement hooks — CONFIRMED, highest severity finding this round

Today's fix closed the `../`-traversal bypass by adding `_wsr_collapse()`, a
purely lexical `.`/`..` normalizer, to `core/hooks/_workspace-route`. It's
explicitly lexical by design ("collapses `.`/`..` segments out of an
absolute path string purely lexically (no filesystem access...)" —
`_workspace-route:73-75`). That same design choice means it — and every
check built on top of it — never resolves symlinks. Directly reproduced
three ways against the current code:

1. **`phase-gate` containment escape.** `phase-gate:89-90`'s `check_target`
   allows any `abs_path` matching `"$task_dir"/*`, purely by string prefix.
   Created `work/T1/escape -> /tmp/spine-outside-target` (a symlink *inside*
   the task folder, pointing *outside* the project entirely), then sent a
   `Write` to `work/T1/escape/pwned.txt`. `phase-gate` exited `0` — allowed —
   during the `research` phase, when the whole point of this hook is "only
   artifacts inside the task folder may be written."
2. **`path-escalate` protected-path escape.** `path-escalate:86`'s
   `check_target` matches `$rel` against the glob from
   `.spine/protected-paths.conf`. With `src/auth/**` protected, created
   `src/other/notprotected.txt -> src/auth/secret.txt` (a symlink at an
   *unprotected*-looking name, pointing *into* the protected tree), then
   sent a `Write` to `src/other/notprotected.txt`. `path-escalate` exited
   `0` — allowed — at Class 0, when the direct path `src/auth/secret.txt`
   is correctly denied (exit 2) by the same hook.
3. **`dep-gate` manifest escape.** Same shape against
   `dep-gate:69`'s `#manifest`-tagged glob match: a symlink named
   `pkg/notmanifest.json` pointing at a real manifest path caused `dep-gate`
   to exit `0` with no `ask` prompt, versus the correct `ask` JSON emitted
   for the direct path.

**Root cause is compounded, not just latent, by a second gap:** `ln` is
completely absent from `core/hooks/_bash-write-targets`'s fast-path
detection regex (`_bash-write-targets:75` —
`'(^|[^0-9&])[0-9]*>{1,2}|\bsed\b.*-i|\bperl\b.*-i|\btee\b|\bcp\b|\bmv\b|\binstall\b|\btruncate\b|\bdd\b...'`
has no `ln` alternative). Reproduced: `bash_write_targets 'ln -sfn /etc/passwd
work/T1/escape'` prints nothing at all — not even `UNRESOLVED:` — so
`phase-gate`/`path-escalate` don't merely fail to deny an `ln -s` call, they
never even notice it happened. This means the whole exploit chain requires
no pre-planted symlink and no Edit/Write trickery: a single Bash call,
`ln -s /etc/passwd work/T1/x`, during `research`/`plan` phase is silently
unchecked, and a subsequent `Write` to `work/T1/x` then passes `phase-gate`
cleanly (finding 1, above) — full write access to anywhere the process can
reach, from the very first phase, when this hook's entire purpose is to
confine writes to the task folder.

The hook-side bypass in all three cases is CONFIRMED by direct reproduction
against the actual scripts. Whether Claude Code's own `Write`/`Edit` tool
implementation follows a symlink at the OS level the same way a shell
redirect does is PLAUSIBLE, not independently verified end-to-end through
the real tool (only through the hook scripts directly, called the same way
Claude Code's PreToolUse mechanism calls them) — but POSIX write semantics
almost universally follow symlinks by default, so this is a reasonable
default assumption, not a stretch.

This is the same *class* of bug the fix shipped today closed (lexical-only
path handling with no ground-truth filesystem resolution), via a different
*mechanism* the fix didn't address. It undermines `docs/tradeoffs.md`'s own
framing that "one layer is a real hook; everything else is a script an
agent is instructed to act on" — the real hook layer itself has a gap wide
enough to walk through.

**Minimal fix:** resolve symlinks before comparing — `abs_path="$(cd
"$(dirname "$abs_path")" 2>/dev/null && pwd -P)/$(basename "$abs_path")"`-
style physical resolution (or a `realpath -q` call, already used
successfully elsewhere in this codebase at `core/scripts/setup:204`) for any
path component that already exists on disk, applied in
`_workspace-route`'s `abs_path` computation and reused by all three hooks;
plus add `ln` to `_bash-write-targets`'s fast-path regex and a target
extractor (final non-flag argument, same shape as `cp`/`mv`).

**Fixed:** added `_wsr_realpath()` to `core/hooks/_workspace-route` — a
bash-3.2-compatible physical resolver (follows an existing symlink,
bounded against a cycle; canonicalizes an existing ancestor directory via
`cd -P`/`pwd -P`; leaves a not-yet-existing final component as-is, nothing
to resolve). Ownership determination (which root's `.spine/` governs a
write) stays based on a root-alias-normalized-but-otherwise-lexical path —
resolving it fully would have wrongly granted the disclosed
out-of-project allowance to exactly the symlink-escape case this fix
exists to catch; only `WSR_REL` (what protected-path globs and task_dir
containment actually compare against) uses the fully resolved form.
`_bash-write-targets` gained `ln` in its fast-path regex plus a target
extractor mirroring `cp`/`mv`. Verified with direct repro, both fixed
forms (symlink pre-planted; `ln -s` issued live via Bash) across all
three hooks, in both single-repo and workspace mode, under both a
canonical path and one traversing a real symlinked ancestor (macOS's
`/tmp` → `/private/tmp`) — the latter surfaced a genuine regression
mid-fix (ownership comparison used an unresolved `$project` against a
resolved file path, or vice versa, depending on which side got touched
first) that a scratchpad-only test would have missed; fixed by resolving
the project root once (`$project_res`) and normalizing every comparison
consistently against it. `core-selftest` green throughout.

---

## Scripts — `core/scripts/*`

### 1. `smoke_in_floor` profile setting is documented, validated, shipped in the `prototype` preset — and `floor` never reads it. CONFIRMED, reproduced.

`core/scripts/profile-check:34,50-51` validates it; `core/templates/profile.json:4`
ships it; `core/ADAPTER-CONTRACT.md:468,719,730,747` documents it governing
whether smoke joins the Class 1 floor, with "the team profile wins" stated
explicitly under Precedence. `core/scripts/floor` has zero references to
`profile.json` (`grep -n profile core/scripts/floor` — no hits); its Class 1
smoke-inclusion decision (`floor:342-366`) only consults `smoke-run`'s
capability status and the personal runtime-budget cache. Reproduced directly
with `profile.json: {"smoke_in_floor": false}`, smoke implemented and cached
under budget: `floor 1` still ran and passed smoke-seed/run/golden anyway. A
team on the `prototype` preset — the one preset that exists specifically to
keep Class 1 cheap — pays smoke ceremony on every Class 1 task regardless.

**Fix:** in `floor`, read `.spine/profile.json`'s `smoke_in_floor` (default
`true`) before the Class 1 `include_smoke` block and short-circuit to 0 when
false.

**Fixed:** added the read; used the `jq '.smoke_in_floor | if . == null
then true else . end'` idiom, not `// true` — the latter treats a
present-but-`false` value as falsy too (jq's `//` gotcha), which would
have silently coerced an explicit opt-out back to the default and shipped
a fix that doesn't fix anything. Added permanent `core-selftest`
regression coverage (3 new cases: default-true, explicit-false,
explicit-true) that reproduces this exact bug class if it ever regresses;
verified green.

### 2. Re-dispatched adversary tokens either overwrite or go unread. CONFIRMED, seen in real production data.

`verify/SKILL.md`'s fix-and-reverify flow re-dispatches `falsifier`/
`security` within one `/verify` pass, but the harvest instruction
(`verify/SKILL.md:341`) only ever names the same phase key
(`ledger harvest <task-id> falsifier ...`), and `ledger`'s harvest case
(`core/scripts/ledger:127`, `.phases[$phase].tokens = $u`) is a plain
overwrite — a second harvest under the same key silently discards round 1's
cost. Real evidence: `/Users/michaelbart/tgml/work/20260821-verdict-menu-dropdown/ledger.json`
has phase keys `falsifier`, `security`, and a hand-invented
`falsifier-round2` — a real session dodged the overwrite bug this way. But
`render-task`'s `phase_source_keys()` (`render-task:180-193`) is a fixed
case statement (`verify) echo "verify falsifier security" ;;`) that doesn't
know about `falsifier-round2`, so that task's ~101.8k tokens are invisible
in its own rendered per-task report, even though `ledger aggregate`'s
generic fold over `.phases` (`ledger:250-284`) does count them correctly
cross-task. Either path a session takes — same key or invented key — some
reader under-reports real spend for that task.

**Fix:** make `ledger harvest` additive on an existing `tokens` value, or
standardize a `-round<n>` suffix and change `render-task`'s phase-key
matching to a prefix match.

**Fixed: both.** `ledger harvest` now sums onto an existing phase's
`tokens` instead of overwriting, so `verify/SKILL.md`'s literal
same-key-every-round instruction is correct on its own — verified with a
synthetic two-round harvest (input/output/cache totals summed correctly).
`render-task`'s `tokens_for()` also changed from an exact-key list to a
prefix match (`"falsifier"` now also matches a real `"falsifier-round2"`
key) for backward compatibility with already-shipped ledger.json files —
verified against the exact real ledger this finding cites
(`tgml/work/20260821-verdict-menu-dropdown/ledger.json`): the previously-
invisible ~101.8k tokens now fold into the rendered total (1,205,589 →
1,307,377), read-only, no mutation to that real project's file. The jq
scoping for the prefix match needed a second pass — piping into
`startswith(. + "-")` silently rebound `.` to the piped value instead of
the outer element, always testing a string against itself; fixed with an
explicit `any(...; . as $name | ...)` bind.

### 3. `render-dashboard` never picked up two fields the same-day fix added elsewhere. CONFIRMED.

Today's fix wired `avg_conformance_score`/`second_approver_count` into
`ledger aggregate`, `render-task`, and `costs/SKILL.md`, but not into
`render-dashboard`, which independently calls `ledger aggregate`
(`render-dashboard:439`) and renders its own "Cost & drift instrument" stat
grid alongside sibling stats `bypass_count`/`hand_tracked_task_count`/
`tooling_gap_count`/`avg_deviation_count` (`render-dashboard:1045-1060`, and
the by-engineer table at `:1079`) — grep confirms zero references to either
new field anywhere in the file. Rendered against `/Users/michaelbart/tgml`'s
19-task history: the drift section shows 7 stats, missing exactly these
two, in the one place a human dashboard-reader would expect them next to
the other cross-task quality signals. Same bug class the fix itself was
meant to close, missed for this one reader.

**Fix:** two more stat entries plus two by-engineer columns, mirroring the
existing pattern at `render-dashboard:1045-1060,1079`.

**Fixed:** both added. Verified by rendering `render-dashboard` against
`/Users/michaelbart/tgml`'s real ledger history (read-only) — the stat
grid now shows "avg conformance score" (0.63) and "second-approver
reviews" (8) alongside the existing cross-task signals. The by-engineer
table's new columns were verified against a synthetic multi-engineer
fixture (tgml has only one real engineer, so its own history doesn't
exercise that path) — including the `avg_conformance_score: null` case,
which renders as `—` rather than crashing on the awk numeric format.

### 4. `render-task` can crash silently (exit 0, no report) on the very field the same-day fix introduced. CONFIRMED, reproduced twice.

`render-task:656,660` do `[[ "$plan_lines" -gt 200 ]]` / `[[ "$briefing_lines"
-gt 60 ]]` with no validation that the ledger value is a clean integer.
Neither `task/SKILL.md:409` nor `ship/SKILL.md:481` specifies `wc -l <file`
vs. `wc -l file` when instructing the count to be recorded — the latter
(quite natural to type) prints `"<n> <filename>"`, which `ledger set` stores
verbatim as a string when it isn't valid JSON (`ledger:139-143`). Reproduced
against this machine's actual bash 3.2.57 (the version this codebase
explicitly targets, e.g. `contract-touch:16-18`): with `plan_line_count` set
to `"42 work/task1/plan.md"`, `render-task` dies mid-render at the `-gt`
comparison (`work: unbound variable` under `set -uo pipefail`'s nounset) —
**exit code 0**, no `report.html` written, no error surfaced.
`task-report/SKILL.md:27-32` only checks that the script produced an output
path, "never a gate" — so a caller following that instruction would believe
the render succeeded.

**Fix:** guard both comparisons with a digits-only regex check first
(`[[ "$plan_lines" =~ ^[0-9]+$ ]] && [[ "$plan_lines" -gt 200 ]]`), falling
back to rendering without the cap check on a non-numeric value instead of
aborting.

**Fixed:** guard added for both fields; reproduced the exact crash
(`plan_line_count: "42 work/task1/plan.md"`) against real bash 3.2.57
before the fix (dies mid-render, exit 0, no output) and confirmed it now
renders cleanly with the raw value shown but no (nonsensical) cap
judgment. Also fixed the root cause in `task/SKILL.md` and `ship/SKILL.md`:
both now say `wc -l < file` explicitly, not `wc -l file`, so a session
following the instruction literally can't produce the malformed value in
the first place.

### 5. Two more written-but-never-read ledger fields — same bug class the same-day fix targeted, missed. CONFIRMED.

- `class_declared` (`task/SKILL.md:686`) — `render-task:87-90` explicitly
  documents avoiding it ("seen null in real fixtures even when the class
  file itself held the real value"), and neither `ledger aggregate` nor
  `render-dashboard` nor `costs/SKILL.md` reads it. Fully dead.
- `pr_description` (`ship/SKILL.md:573`), written with the explicit stated
  intent "so a future `/costs` view can notice a ship that skipped this
  step" — `costs/SKILL.md` never mentions it, and no script reads it
  anywhere. Identical failure shape to the `ship_time_regrounding_floor`/
  `_index` false-claim the same-day fix already corrected once.

(Checked as a control: `class_downgraded_from`, also written by
`task/SKILL.md:264`, genuinely is read — `costs/SKILL.md:97-101` counts
`work/*/ledger.json` files with it set. Not every field written today is
dead — just these two, plus `spec_hash` below in templates.)

**Fix:** drop `class_declared` and `pr_description` from what's written, or
give each a real reader (`pr_description_count` in `ledger aggregate`,
mirroring `second_approver_count`'s shape).

**Fixed: one of each.** `class_declared` dropped entirely — removed from
`ledger init`'s schema and from `task/SKILL.md`'s write instructions
(it was also unreliable, per `render-task`'s own comment, so fixing its
reader wasn't the right call). `pr_description` kept and given a real
reader: `pr_description_count` added to `ledger aggregate` (both plain
and `--by-engineer`), `costs/SKILL.md` updated to report it, and
`ship/SKILL.md`'s comment corrected to name the now-true consumer.
Verified against real data (`tgml`: `pr_description_count: 19` of
`task_count: 20`) and a fresh `ledger init` (confirmed `class_declared` no
longer appears in a new ledger.json's key set).

### Minor, not worth a separate fix
`claims-check:53-55`'s header comment says `--diff` mode exits 1 only for
`[UNDECLARED]`; the code (`:175`) exits 1 for any blocking entry including
`[DECLARED]`. Harmless — `ship/SKILL.md:119-123` treats both as halt-tier
regardless of exit code — just a stale comment.

### What holds up well
`ledger`, `floor` (apart from finding 1), `conformance`, `claims-check`,
`check-stale`, `contract-touch`, `decision-hash`, `decision-index`,
`design-gate`, `next-milestone-task`, `profile-check`, `propagate`,
`registry-sync`, `second-approver-check`, `setup`, `ui-touch`,
`verdict-filter`, `adapter-conformance`, `q` — all careful, handle the
bash-3.2 constraint they document, degrade explicitly rather than silently,
matched their own header claims under direct testing against both synthetic
fixtures and real installed-project data.

---

## Skills — `core/skills/*`

### 1. `${CLAUDE_SKILL_DIR}` lexical-collapse warning exists in 3 of 17 skills that need it. CONFIRMED.

`task/SKILL.md:199-207`, `spine/SKILL.md:16-22`, `intake/SKILL.md:17-23`
each carry a ~6-line warning: don't let the model textually collapse
`skills/<name>/../..` to `.claude/`, because `.claude/skills/<name>` is a
real symlink into the checkout (`core/scripts/setup:200-228` confirms this)
and a lexical collapse yields a nonexistent path. `ship/SKILL.md` uses the
identical `${CLAUDE_SKILL_DIR}/../../scripts/<name>` pattern **15 times**
(more than `task`'s 13), `design` 10, `verify` 8, `bootstrap` 7, `workspace`
5, `adopt`/`intake` 4 — none of these carry the warning. `update/SKILL.md`
sidesteps the problem with a different, more robust `readlink -f`
convention, applied nowhere else.

Scenario: `/ship` — the merge gate, second-approver check, bypass logic —
hits this exact failure mode on any of its 15 script invocations with no
instruction telling the model what went wrong or how to avoid triggering
it, at exactly the highest-stakes, most `disable-model-invocation`-guarded
command in the system.

**Fix:** add the same ~6-line warning to `ship` and `design` at minimum
(heaviest users after `task`); ideally all 14 skills missing it.

**Fixed: all 13** (of the 14 named — `update/SKILL.md` turned out to
already sidestep the problem via `readlink -f`, confirmed by inspection,
so it didn't need one). Heavy users (`ship`, `design`, `verify`,
`bootstrap`, `workspace`, `adopt`, `costs`, `roadmap`) got the fuller
warning matching `task/SKILL.md`'s own; the five single-occurrence skills
(`ratchet`, `remap`, `roadmap`, `tasks`, `task-report`, `visualize`) got a
terser one-clause version at their one `${CLAUDE_SKILL_DIR}` use, sized to
the actual exposure rather than uniformly copy-pasting the full block —
token cost proportional to risk. Verified by grepping every skill for
`CLAUDE_SKILL_DIR` usage vs. warning presence (a line-wrap-tolerant
substring check, since the warning text itself wraps across lines the
same way the audit's own rule 8 warns about).

### 2. `/verify`'s adversary re-run caching heuristic is 204 lines of unbacked prose. CONFIRMED.

`verify/SKILL.md:194-395` (step 3, "Run the adversaries") is 204 of 486
lines — 42% of the file — instructing the agent to hand-compute sha256
content hashes, union several file sets, compare against a prior
`<agent>-coverage.json`, and check severity thresholds, by hand, on every
single `/verify` call including the common first-dispatch case where none
of it applies yet. `grep -rl "coverage.json\|content hash\|REUSED"
core/scripts/` finds no backing script — `check-stale` and `decision-hash`
do comparable content-hash work but as real scripts, not prose the model
re-derives every time. Same shape as the line-cap gap the same-day audit
fixed (self-reported instruction, no mechanical backstop), except here it's
the caching mechanism's own correctness that's unbacked, and the token cost
is paid specifically to save cost — a poor trade if the hand-computation
itself is expensive or gets it wrong.

**Fix:** a small `core/scripts/adversary-cache-tier <task-id> <agent>`
script that outputs `REUSED`/`FOCUSED <files>`/`FULL` mechanically, cutting
the 204 lines to roughly the length of the `check-stale` call site
elsewhere in the same file.

**Fixed:** built `core/scripts/adversary-cache-tier` exactly as scoped —
reads a blast-radius file list from stdin, hashes each against
`<agent>-coverage.json`, checks the prior verdict's severity ceiling,
prints one of `REUSED`/`FOCUSED`/`FULL` mechanically. `verify/SKILL.md`'s
step 3 rewritten to call it instead of hand-deriving the tier (486 → 445
total lines, net, even after also adding the CLAUDE_SKILL_DIR warning
above). Added 8 `core-selftest` cases exercising every branch (no
coverage file, full/partial/zero match × none/low/medium/high severity) —
**this caught a real bug in the new script**: its stdin-reading loop
(`while IFS= read -r line; do`) silently dropped the last blast-radius
file whenever the input had no trailing newline, because bash's `read`
returns non-zero on a final unterminated line even though it populates
the variable. Fixed with the `|| [[ -n "$line" ]]` idiom already used
elsewhere in this codebase (`_bash-write-targets`). Exactly the kind of
bug this audit's own methodology (write the test, actually run it) exists
to catch — it would not have been found by review alone.

### 3. Real-world spot check: the 60-line briefing cap is already exceeded with no visible consequence. Matches disclosed tradeoff, not a new finding.
`/Users/michaelbart/bookmarks/work/20260808-walking-skeleton-items-auth/briefing.md`
is 90 lines. This task predates today's line-count-recording fix (no
`briefing_line_count` in its `ledger.json`) — confirms the fix's own
"visibility, not enforcement" caveat is real in practice, not just
hypothetical.

### What holds up well
`costs/SKILL.md`'s entire documented field list traced correctly against
`ledger`/`conformance`/`render-task`'s real output. `next-milestone-task`'s
four-way branch matches the script exactly. `core-selftest` green.
`workspace/SKILL.md`'s §0/§6 heading contradiction from the prior round is
genuinely fixed and internally consistent. `adopt/SKILL.md` avoids the
falsifier/security duplication trap by cross-referencing `bootstrap`'s
prose instead of restating it — a good pattern worth reusing anywhere else
a near-duplicate skill pair shows up. Live artifact
`/Users/michaelbart/tgml/work/20260822-category-state-sealing-trigger/briefing.md`
spot-checked against all seven `writing-mandate.md` rules — genuinely
followed, not just asserted. `spine/SKILL.md`, `tasks/SKILL.md` are
genuinely read-only; `security-checklist/SKILL.md` (61 lines) is tight with
no bloat.

---

## Agents, templates, rules, ADAPTER-CONTRACT, docs

### 1. `spec_hash` in `workspace.json` is dead, write-only data — and has already drifted in real usage. CONFIRMED.
`core/templates/workspace.json:15` calls it "contract-touch's own
staleness/drift signal." `core/scripts/contract-touch` never reads it
(zero grep hits) — it classifies additive/breaking straight from `git diff`
against `spec_path`. `workspace/SKILL.md:160` only requires preserving it
byte-for-byte on rewrite. Real evidence:
`/Users/michaelbart/bookmarks-workspace/workspace.json`'s registered
`spec_hash` no longer matches `contracts/items-api/spec.md`'s real sha256,
and nothing notices. Same bug class as the ledger fields fixed today,
missed here because it's a template/workspace field rather than a ledger
one.

**Fix:** either wire it into `contract-touch`'s staleness check, or drop the
field and its misleading comment.

**Fixed: dropped.** `contract-touch`'s real mechanism (`git diff` against
`spec_path`) is strictly stronger than a hash comparison would be (no
manual re-hash-and-re-register bookkeeping needed on every legitimate
spec edit) — wiring `spec_hash` in would have been redundant, not
additive. Removed from the template, its comment, and
`workspace/SKILL.md`'s byte-for-byte preservation list. Left the real
installed project's (`bookmarks-workspace`) already-stale `spec_hash`
value untouched — that's a different repo, out of scope for a spine-core
fix.

### 2. `/task-report` and `/visualize` are the only two of 17 skills missing `disable-model-invocation: true`. CONFIRMED.
README.md:116-117 states "most take `disable-model-invocation: true` — you
type them, the model doesn't reach for one on its own"; every other
read-only command (`/spine`, `/tasks`, `/costs`, `/remap`, `/ratchet`,
`/update`) already has it. Nothing distinguishes these two — looks like an
oversight on the two most recently added commands.

**Fix:** add the frontmatter key to both.

**Fixed:** added to both.

### 3. `briefing.md`'s "stable, so it's grep-able" claim is contradicted by the system's own oldest real output, with no migration path. CONFIRMED.
`core/templates/briefing.md:15-21` says the bold-label convention (`**Floor:**`,
`**Overrides & bypasses:**`) is "kept stable across tasks... it's what makes
this file grep-able the day something starts reading briefings in
aggregate." Real evidence: `/Users/michaelbart/bookmarks/work/20260808-walking-skeleton-items-auth/briefing.md`
and `.../20260809-item-date-range-filter/briefing.md` use an entirely
different, older heading structure (`## What changed`, `## Why`, `## Deviations
taken`) with no bold labels — while `tgml`'s more recent briefings use the
current shape correctly. No backfill/migration mechanism exists for
already-shipped artifacts when the template changes, so a future aggregate
reader would silently miss or misparse these two real, already-shipped
tasks — exactly what the "stable, so it's safe to rely on" rationale claims
won't happen. Low severity (2 affected tasks in the samples checked) but
worth a one-line disclosure in `docs/tradeoffs.md`'s "Known limits" rather
than an unqualified stability claim.

**Fixed:** added the disclosure to both `docs/tradeoffs.md`'s "Known
limits" and a parenthetical in `briefing.md`'s own header comment — the
stability guarantee is prospective-only, doesn't retroactively rewrite
older shipped briefings. Not a mechanical fix (there's no reader to fix
yet, since "no script or skill reads a briefing.md back in" today) — this
is purely closing the gap between the claim and reality until an
aggregate reader is actually built, at which point it needs to tolerate
the older shape too.

### What holds up well
All four `core/agents/*.md` files' claimed tool permissions/isolation match
actual invocation (falsifier's `isolation: worktree` + Edit/Write is real
and correctly wired via `verify/SKILL.md:321`). `core/rules/contracts.md`
and `migrations.md` check out exactly against `contract-touch` and
`ADAPTER-CONTRACT.md` §2.2/§3.7. Current-shape `research.md`/`plan.md`/
`verify.md`/`briefing.md` templates produce genuinely good real output —
read several full `tgml` examples end-to-end as a human reviewer would:
dense, no boilerplate, real file:line citations, an honestly-reported
falsifier-caught bug and the decision distilled from it
(`20260821-filmrow-navigation`, read in full this round — see below).
`hook-guard`'s fail-closed behavior is real and correctly wired into every
real install checked. `docs/tradeoffs.md` spot-checks (bash-write-target
patterns, `standard` preset's `autonomy_ceiling: auto` default, Class 2
smoke unreachability) all matched current code — no fresh staleness found
in the samples this round checked, beyond finding 3 above.

---

## Real-usage read: is this actually digestible?

Read `/Users/michaelbart/tgml/work/20260821-filmrow-navigation/{plan,verify,briefing}.md`
end to end, the way an engineer would. `plan.md` (109 lines): states the
fix, the rejected alternative and *why* it was rejected (an M3 milestone
already plans to rebuild that exact chrome), decide-alone vs. halt lines
that are genuinely specific to this diff rather than boilerplate, three
numbered steps each with a concrete browser-testable acceptance check.
`verify.md` (80 lines): a real adversary-caught bug (`stopPropagation`
doesn't reliably block `next/link`'s own navigation), the fix, a second
adversary round re-attacking the fix fresh, one deliberately-accepted
low-severity tradeoff named and justified rather than silently absorbed,
and an honest capability-gap disclosure (no UI-click simulation exists in
this project's smoke harness). `briefing.md` (17 lines): bottom-line-first,
names the surprise plainly, points at the distilled decision record. This
is the system working exactly as designed — scannable, no checklist
padding, and it caught a real bug a human skim of the diff plausibly would
have missed (a timing-dependent event-bubbling defect). Line counts across
the 8 most recent `tgml` tasks sampled: `plan.md` 54–109 lines (cap 200),
`briefing.md` 17–22 lines (cap 60), all comfortably under. This is real,
not theoretical, evidence the digestibility goal is being met in the
common case.

---

## What's verified working well — don't touch

- The `research → plan → implement → verify → ship` artifact trail, judged
  against real, multi-task production history (`tgml`, 29 tasks), is
  genuinely scannable and catches real bugs. Not a paper process.
- `core-selftest` passes cleanly; the `../`-traversal collapse fix and the
  Class-2 migrate-rehearse regression fix from today both hold under direct
  re-testing.
- The floor's degrade-rather-than-silently-pass posture, `claims-check`,
  `check-stale`, `contract-touch`, and the multi-repo workspace routing
  logic are careful and internally consistent, including correct bash 3.2
  handling throughout.
- The adversary agents' actual tool permissions and isolation match what
  they claim (falsifier's worktree mutation mandate is real, not aspirational).
- `docs/tradeoffs.md` remains, on the sample checked this round, an honest
  and largely current account of the system's known limits — the one drift
  found (`spec_hash`, `briefing.md`'s stability claim) is narrow, not
  systemic.

---

## Overall verdict

**Spine needs real, non-hypothetical work — but it is not overbuilt, and
the core loop is earning its complexity.** The strongest evidence for that
second half is the `tgml` production history: 29 real tasks, artifacts that
stay inside their own caps, and at least one adversary-caught bug
(`filmrow-navigation`) that shipped correctly *because* the falsifier
existed, not despite it. This is not a system rediscovering problems that
don't come up in practice.

The strongest evidence for the first half is the symlink bypass: the one
enforcement tier this system calls "a real hook, not just an instruction"
has a gap wide enough to defeat all three PreToolUse gates, discovered
*immediately* after the structurally identical `../`-traversal bug was
fixed in the same files, in the same session, by the same kind of audit.
That's not an isolated slip — it's a pattern (lexical-only path handling
papering over a filesystem-level guarantee) that a single symlink-aware fix
closes for good, and it should happen before the `../` fix is trusted as
"done." Layered onto that: a now-second occurrence of "field written,
never wired to a reader" (`spec_hash`, `class_declared`, `pr_description`,
on top of the six the same-day fix already caught) suggests this bug class
won't stop recurring under manual audit alone — a cheap, mechanical check
(grep every `ledger set`/config-field write site, confirm at least one
reader) would catch it structurally instead of needing another full sweep
every time.

**What would change this verdict:** if the symlink fix lands and a
lightweight "every written field has a confirmed reader" check gets added
to `core-selftest` (so this class of bug can't silently reappear a third
time), spine moves from "sound design, real enforcement gaps" to
"in good shape" — the artifact-trail and adversary-review core clearly
already works in practice; what's missing is closing the gap between what
the enforcement layer claims to guarantee and what it actually does.

**Both landed in this round's fix pass.** The symlink bypass is closed
(`_wsr_realpath`, verified against direct repro in single-repo and
workspace mode, under both a canonical path and one traversing a real
symlinked ancestor). The field-coverage check exists
(`core/scripts/ledger-field-audit`) — deliberately *not* wired into
`core-selftest` as a hard pass/fail, since it's grep-based and can false-
positive; it's a periodic manual check in the spirit of `/ratchet`, not
another gate every task pays for. Given that, the verdict above should
read as **upgraded to "in good shape,"** with the caveat that both fixes
are hours old, not battle-tested across many real tasks the way the
`../`-traversal fix from the prior round already has been by the time of
this writing — the honest position is "in good shape, pending the same
kind of real-usage confirmation the rest of this system has earned."

---

## Process note for next time

Round 1's own process note said: run `core-selftest` before and after any
change to `floor`'s gating logic. This round generalizes that lesson,
because it caught two more of its own bugs the same way, in code written
*during this same fix pass*:

1. The hooks fix's first draft (`_workspace-route`'s `abs_path`/`project`
   resolution) passed every repro test run against a scratchpad path — and
   still had a real bug, because every scratchpad test happened to live
   under an already-canonical path. Only re-testing against a path that
   traverses a real symlinked system alias (macOS's `/tmp` →
   `/private/tmp`) surfaced it. **A path-handling fix isn't verified until
   it's tested against at least one path that goes through a real,
   ordinary symlink somewhere in its own ancestor chain — not just the
   adversarial symlink the fix targets.**
2. `adversary-cache-tier`'s stdin-reading loop, and separately
   `ledger-field-audit`'s own "no reader found" branch, both had a
   textbook bash gotcha (`while read` silently drops an unterminated last
   line; `set -e` silently kills a script on a intentionally-nonzero `grep
   -l` with no matches) that only showed up once real `core-selftest`
   coverage or a deliberate negative-case test was written and actually
   run — not from reading the script back.

Neither bug was found by writing the fix carefully and reasoning about
it — both were found by writing a test that exercises the fix and running
it. That's not a new lesson so much as this round's own instance of the
same one round 1 named: **a script is trusted once it's been run against
a case designed to break it, never before.**
