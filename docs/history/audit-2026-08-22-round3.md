# Audit — 2026-08-22, round 3

Independent, adversarial audit run against `main` at commit `14cadd4`, with
round 2's fixes present only as an uncommitted working-tree diff at the
time this round started (see "Governance" below — that was itself this
round's first finding, before any subsystem review began).

Findings below were produced by five parallel subagents (skills; hooks;
scripts; agents/rules/templates/ADAPTER-CONTRACT; docs/README plus a
deliberately broad real-usage read), each independently verifying claims
against real code execution and, where possible, real installed-project
data (`/Users/michaelbart/tgml`, 20 tasks; `/Users/michaelbart/bookmarks`;
`/Users/michaelbart/bookmarks-workspace`, multi-repo; `/Users/michaelbart/
bgr`, `/Users/michaelbart/bookmarks-cli`, `/Users/michaelbart/horizon`).
Every finding was then reviewed and fixed in a separate pass, recorded
inline as it was in rounds 1 and 2 — each "**Fixed:**" note says what was
done and how it was verified, including a negative control (temporarily
reverting the fix and confirming the new regression test actually fails)
for the two hooks fixes, per the lesson round 2 itself named: a test that
passes once a fix is in place isn't evidence unless it's also been shown
to fail without the fix.

---

## Governance

### Round 2's entire fix pass, including its own symlink-bypass closure, was uncommitted — CONFIRMED

`git status` at the start of this round showed 20 modified files and 3
untracked files (including `docs/audit-2026-08-22-round2.md` itself)
against HEAD `14cadd4`. Every "Fixed:" claim in round 2's own report
existed only as a working-tree diff. A fresh clone of this repo at the
time would have inherited none of it, including the highest-severity
finding of that round.

**Fixed:** not a code fix — this is a process note. Everything in this
round's own fix pass, plus round 2's pending diff, should be committed
together once reviewed. (Left to the human running this audit to commit,
per this project's own norm of not auto-committing audit fixes.)

---

## Hooks — `core/hooks/*`

### An absolute `file_path` reaching the project through a different real filesystem alias than `$project` bypassed all three hooks — CONFIRMED, highest severity

`core/hooks/_workspace-route`'s ownership decision (`workspace_route()`)
matched `abs_path` against `"$project_res"/*` (or a member repo's root)
*lexically* — the one place a fully-resolved form (`abs_path_res`) existed,
it was computed but never consulted as a fallback for the ownership
decision itself, only for the relative-path computation once ownership was
already decided. When `CLAUDE_PROJECT_DIR` was given canonically
(`/private/tmp/proj`) but a tool's `file_path` was given through the
ordinary, real macOS `/tmp` → `/private/tmp` alias (same directory), the
lexical match failed, ownership fell through to `(unknown)`, and every
caller's `[[ -n "$WSR_OWNER_ROOT" ]] || return 0` allowed the write —
confirmed for `phase-gate`, `path-escalate`, and `dep-gate`, via both
Edit/Write and Bash. This is the third distinct mechanism (after round 1's
`../` and round 2's symlink resolution) producing the same underlying
failure: a path-comparison guarantee implemented ad hoc per call site
instead of through one consistent normalization step.

**Fixed:** added `_wsr_owns()` to `_workspace-route` — checks the lexical
form first (preserves existing behavior byte-for-byte), and falls back to
checking the fully-resolved form against an already-canonical root only
when the lexical check fails. This closes the alias gap without reopening
the symlink-escape case round 2 closed (a path lexically inside the
project that symlinks its way *outside* it): that case's resolved form
lands outside `$project_res`, so the fallback still correctly reports
`(unknown)`. Verified against the exact repro (alias-mismatch write,
Edit/Write and Bash, all three hooks) plus a full regression matrix
(traversal, planted symlink, unrelated external path, reverse-alias
direction) with no change in any previously-correct decision. A permanent
regression case (`core/scripts/core-selftest`, "hooks:" section) now
exercises this via a self-created symlinked project-root alias (portable —
doesn't depend on the host's own `/tmp` layout, but exercises the identical
code path); confirmed by negative control that this case fails when the
fix is reverted.

### `dep-gate`'s Bash branch never checked `#manifest`-tagged protected paths at all — CONFIRMED

Unlike `phase-gate`/`path-escalate`, whose Bash branches both source
`_bash-write-targets` to extract write targets and re-check them,
`dep-gate`'s Bash branch only regex-matched the raw command string against
`install-command-patterns.conf`. A Bash redirect, `sed -i`, or `cp` onto a
`#manifest`-tagged path (e.g. a package manifest) went through silently,
with no `ask`, while the identical target via Edit/Write correctly forced
one.

**Fixed:** factored the existing manifest-glob check into
`check_manifest_target()`, called from both the Edit/Write branch (as
before) and a new Bash branch that sources `_bash-write-targets` the same
way the other two hooks do, routing every extracted `PATH:` target through
the same check and treating `UNRESOLVED:` as an `ask` (dep-gate's own
"force a human decision when uncertain" posture, mirroring the other two
hooks' fail-closed-to-deny). Verified via direct repro (Bash redirect and
`sed -i` onto a manifest path, both now correctly `ask`) and the new
`core-selftest` cases.

### Zero automated regression coverage existed for any hook — CONFIRMED

Every verification claim for this subsystem, across all three audit
rounds, was one-off manual shell testing. Nothing prevented a silent
regression of any of the three now-closed bugs.

**Fixed:** added 11 direct-repro cases to `core/scripts/core-selftest`
(new "hooks:" section) covering: canonical-path control, the alias-mismatch
case for all three hooks (Edit/Write and Bash), a manifest write via Bash
`sed -i`, an unrelated-external-path control, the reverse alias direction,
a `../` traversal, and a planted symlink into a protected path. Confirmed
meaningful (not just trivially passing) by a negative control: stashing
the `_workspace-route`/`dep-gate` fixes and re-running the suite reproduces
7 failures in exactly the alias-related cases, with the unrelated controls
still passing.

---

## Scripts — `core/scripts/*`

### `verify`'s cost silently vanished from 65% of real single-task reports — CONFIRMED, real production data

`task/SKILL.md` §5 documents `ledger mark <task-id> verify` (which, per §6,
is what harvests `implement`'s token window), but real sessions don't
reliably do it — a sample of 20 real tasks in `/Users/michaelbart/tgml`
showed 13 (65%) missing `.phases.implement.tokens`. `render-task`'s
per-phase nav then showed `verify: not reached` for tasks that plainly
underwent real, sometimes-expensive adversary review (falsifier/security
tokens recorded, just not rolled up under the umbrella phase), since it
only ever displayed cost when `marked_at` was present.

**Fixed (partial, disclosed as a remaining honor-system limit — see
`docs/tradeoffs.md`):** `render-task` now shows the already-computed
subagent cost (`falsifier`/`security`) even when the umbrella phase was
never marked, with an honest "not reached (N tok recorded under a subagent
phase)" label instead of silently dropping it. This does not fix the root
cause — sessions still aren't mechanically forced to call `ledger mark`;
that's now named explicitly in `docs/tradeoffs.md`'s Known limits.

### `ledger-field-audit` had the exact `set -e`/zero-match crash bug its own header claims to already guard against, and a real field was already invisible to it — CONFIRMED

The `fields=` extraction line lacked the `|| true` guard its sibling
`hits=` line has; a zero-match run (no `ledger set <task-id> ...` found
across any skill) would silently kill the whole script under
`set -euo pipefail`. Separately, the line-bound regex already missed a
real, genuinely-read field (`class_downgraded_from` in
`core/skills/task/SKILL.md`, whose reference is split across a line wrap)
— invisible to the tool, neither flagged nor cleared.

**Fixed:** guarded the `fields=` extraction the same way `hits=` already
is; changed extraction to join each file's lines with spaces before
matching, so a line-wrapped `ledger set <task-id>\n<field>` is caught the
same as a same-line one; widened the reader-detection regex to also match
backtick-quoted markdown references (`` `field` ``), which is how
`class_downgraded_from`'s real reader in `costs/SKILL.md` is phrased.
Verified: a synthetic zero-match fixture no longer crashes (clean exit 0,
correct message); a synthetic line-wrapped field is now detected as a
candidate; the real repo now reports `class_downgraded_from` as a
candidate with a found reader (previously invisible either way); full
audit against the real repo still reports clean.

### No permanent test covered floor's Class 2 smoke hard-gate, `ledger`, `render-task`, or `render-dashboard` — CONFIRMED

`core-selftest` doesn't claim to cover the latter three, so this isn't a
false claim, but it meant "core-selftest: PASS" provided zero mechanical
evidence for round 1's headline fix (the ledger-field wiring). Separately,
every Class 2 fixture stubbed smoke as `"implemented"` specifically to
bypass the hard-gate path round 1 fixed, so a silent regression there
wouldn't be caught either.

**Fixed:** added a `core-selftest` case exercising the Class 2 smoke
hard-gate directly (deliberately using a different "not implemented"
status string, `"unavailable"`, than round 1's original repro used,
`"not-applicable"`, per the standing lesson that a fix's own repro isn't
enough to trust it's general). `ledger`/`render-task`/`render-dashboard`
coverage remains a real gap, not addressed this round — flagged for round
4.

---

## Skills — `core/skills/*`

### The multi-repo conformance-score instruction had no multi-repo branch, unlike every neighboring instruction in the same section — CONFIRMED, real production data

`verify/SKILL.md` §4/§5 explicitly branches multi-repo vs. single-repo for
floor results, contract conformance, and UI render — but the
`ledger set <task-id> conformance_score` instruction two paragraphs later
reverted to the single-repo-only form. A real multi-repo task in
`bookmarks-workspace` produced three `conformance-<repo>.json` files (F1
0.75, 0.67, 0.13) and recorded a single `conformance_score: 0.67` — one of
the three, apparently picked arbitrarily, the other two silently
discarded; two sibling multi-repo tasks left the field `null` entirely.

**Fixed:** added an explicit multi-repo clause — average the `f1` field
across every edited repo's own `conformance-<repo>.json`.

---

## Agents, rules, templates, ADAPTER-CONTRACT

### ~84 repo-wide dangling citations to "build prompt §N," a document deleted in an earlier commit — CONFIRMED

Commit `05e4886` deleted `work/.build/build-prompt-extensions.md` and
claimed to have cleaned up the resulting dangling citations. A repo-wide
grep found 84 remaining hits across `core/ADAPTER-CONTRACT.md`,
`core/agents/*`, `core/rules/*`, `core/templates/*`, `core/skills/*`,
`core/scripts/*`, and `core/hooks/*` — most citing specific numbered
sections of a file nobody can open anymore. Same bug class round 1 fixed
for `g1-tee-waitlist`, roughly 20x the size, missed because that cleanup
only grepped for one literal dead-doc name.

**Fixed:** for each citation, inlined the rule it pointed at (most already
paraphrased it in adjacent prose) and dropped the dead pointer. `core/hooks
/dep-gate`'s two occurrences were fixed as part of that file's HOOKS-2 fix,
above; the remaining ~82 were fixed as a dedicated, prose-only pass.
Verified: repo-wide grep for `build prompt` (excluding this audit's own
docs) now returns zero hits; `core-selftest` still passes (confirming no
script syntax broke during the prose-only edits).

### The briefing/PR-description line cap measured the wrong unit for its own prose format — CONFIRMED, real production data

`briefing.md` calls itself "≤1 page, hard," operationalized as "≤60
lines" — but the template's one-unwrapped-paragraph-per-label format means
`wc -l` counts each paragraph as one line regardless of length, unlike
`plan.md`'s wrapped-bullet style, where line count actually tracks page
length. All 19 real, shipped briefings in `/Users/michaelbart/tgml`
measured 17–29 lines but 375–805 words (up to ~6,000 characters) — several
already past a fair page's worth of reading material while showing well
under half the "cap."

**Fixed:** `ship/SKILL.md` now also records `wc -w < briefing.md` as
`briefing_word_count`, with ~600 words named as the real over-cap signal
for this format; `render-task` reads it and flags an over-cap briefing by
word count even when the line count looks fine.

### Stale comment in `render-task` described a bug already fixed elsewhere — CONFIRMED, cosmetic

Reworded to past tense; the numeric guard itself was already correct and
unchanged.

### README.md said "most" skills take `disable-model-invocation: true` — CONFIRMED, cosmetic

All 17 user-invocable command skills do. Corrected to "all."

---

## For round 4

Not chased down this round — worth a look next time:

- `ledger`/`render-task`/`render-dashboard` still have no permanent
  `core-selftest` coverage (scripts finding above).
- The verify-phase-marking honor-system gap (scripts finding above) is
  disclosed, not fixed — worth checking whether real production data still
  shows a high miss rate after this round, or whether it's improved.
- The three-hooks bypass pattern has now recurred via four distinct
  mechanisms (`../`, a planted/live symlink, a project-root alias, and
  dep-gate's Bash branch never running the check at all). Worth
  considering whether a fourth round finding a fifth mechanism is a signal
  the fix needs to be structural (one shared, single-call path-resolution
  helper used everywhere) rather than another incremental patch — and
  worth testing forms not yet tried here: a trailing-slash difference, a
  Unicode normalization (NFC/NFD) difference on a case-insensitive/
  normalizing filesystem, and a relative `CLAUDE_PROJECT_DIR` value (every
  test so far, across all three rounds, has used an absolute one).
- `pr-description.md` shares `briefing.md`'s paragraph-per-label format
  but has no line-cap/word-cap tracking mechanism at all — not clear
  whether it needs one (no cap is currently claimed for it), but worth
  confirming that's a deliberate omission and not an oversight.
