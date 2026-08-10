# Extension Build — Phase A Handoff

Date: 2026-08-08. Builder: Claude Code (Sonnet 5), interactive session.
Scope: Spine Extensions build prompt §0 (primitive verification) and the
build plan for §9's Phase B–E. Nothing outside primitive verification and
planning was built this phase — no core files were modified.

Read order for anyone resuming: this file, then `README.md` and
`docs/tradeoffs.md` (already-current, not modified this phase), then the
actual state of `core/` (also unmodified — the `find` output below is
still accurate as of this writing).

## 1. Primitive verification (build prompt §0)

### §0.1 — Hook topology: workspace-root hooks fire by path; added-directory hooks never load. **CONFIRMED, empirically, both halves.**

This is the fact Extension B's entire enforcement model depends on, so it
was tested for real rather than accepted from the build prompt's own
citation (issue #52934 — not independently verified, only the *behavior*
was).

**Method:** my own session's Bash tool cannot invoke `claude` at all —
every attempt (`claude -p ...`, even bare `claude --version`) was denied
outright by this machine's Auto Mode classifier, with or without
`--dangerously-skip-permissions`. This is the exact wall
`docs/tradeoffs.md`'s "Auto Mode classifier wall" section already
documents for spine *script* invocation — it turns out to also block
invoking the `claude` binary itself from within a session, which is new
information beyond what that section currently states (see §3 below). The
engineer ran the test commands directly in their own shell via Claude
Code's `!` passthrough instead, which does not go through my Bash tool's
classifier. Scratch fixtures live at
`/private/tmp/claude-501/-Users-michaelbart/6e10067c-2712-4d52-a0e8-27cb2393b63d/scratchpad/primitive-test/`
(session-scratch, not durable — reproduce from this handoff if needed
later, don't rely on the directory still existing).

**Fixture:** `workspace/.claude/settings.json` wires a hook
(`block-added-writes`) that denies any `Edit`/`Write` whose `file_path`
contains `/added/`. `added/.claude/settings.json` wires a *different* hook
(`block-everything`) that denies unconditionally — chosen to be a maximally
loud signal if it ever fired at all.

**Test (a) — does the workspace hook see and gate a path inside the added
directory?** `cd workspace && claude -p "<edit added/target.txt>" --add-dir
<added> --permission-mode acceptEdits`. Result: the nested session's own
reply was *"A hook is blocking writes under the `added/` directory... I
can't edit target.txt since it lives outside the permitted workspace
root."* — `added/target.txt` confirmed byte-identical before/after
(`unmodified`, unchanged). **The workspace-root hook fired on a target
path inside the added directory.**

**Test (b) — does the added directory's own hook ever fire, even for an
edit that has nothing to do with it?** Same session shape, but the edit
target was `workspace/local.txt` (inside the workspace itself, matching
neither hook's deny condition by content, but within scope of *any* loaded
`Edit|Write` hook, including `block-everything` if it were loaded). Result:
*"Done — replaced unmodified with MODIFIED in local.txt."* — confirmed by
reading the file directly (`MODIFIED`). **`block-everything`, defined only
in `added/.claude/settings.json`, never fired**, despite `added/` being
attached to the session the entire time via `--add-dir`.

**Test (c) — re-run test (a) using `permissions.additionalDirectories` in
`settings.json` instead of the `--add-dir` CLI flag** (this is the actual
mechanism `/workspace` will generate — an engineer starting a normal
session won't type `--add-dir` by hand). Removed `--add-dir`, added
`"permissions": {"additionalDirectories": ["<added-path>"]}` to
`workspace/.claude/settings.json`, re-ran against a fresh target file.
Result: file unchanged — **identical blocking behavior to the CLI flag.**
This is the load-bearing confirmation: the exact settings shape `/workspace`
will write behaves the same as the flag-based test.

**Conclusion:** §0.1 holds exactly as the build prompt states. Extension
B's design — hooks load once from the workspace root, consult
`workspace.json` to route by path, and are never installed per-member-repo
— is sound. **No redesign needed at Phase A.**

### §0.2 — `additionalDirectories`/`--add-dir`: added directories are readable/editable under the session's permission mode. **Confirmed as a side effect of §0.1's own tests** — the nested session could target and (when not hook-blocked) successfully edit a file under `added/`, an absolute path outside its own project root, with only `--add-dir`/`additionalDirectories` granting that reach.

### §0.3 — CLAUDE.md from added directories is env-gated. **Not tested, per the build prompt's own instruction not to build on it.** Extension B does not depend on this — per-repo context reaches sessions through skills reading files (`/workspace`'s design already commits to this). Accepted as documented, unverified, and irrelevant to what's being built.

### §0.4 — Do skills/agents resolve like settings (startup dir + user scope)? **NOT verified this phase — deferred to Phase D, with a concrete test plan.**

Checked `code.claude.com/docs/en/settings` and `.../skills` via WebFetch;
neither page states whether skills or agents from an `additionalDirectories`
entry are discovered. This determines whether `/workspace` needs to
install its own skill/agent symlinks (mirroring `/bootstrap`'s mechanism)
at the workspace root only, or whether member repos' own
`.claude/skills`/`.claude/agents` (if they're independently spine-installed,
which brownfield member repos will be, per Extension B's design) leak into
the workspace session.

**Test plan for Phase D** (uses the now-validated `!`-passthrough technique
from §0.1): place a throwaway skill at
`added/.claude/skills/added-only-skill/SKILL.md` with
`disable-model-invocation: true` and a distinctive one-line reply. From a
`workspace`-rooted session with `added/` attached via
`permissions.additionalDirectories`, invoke `/added-only-skill` — if it
resolves and runs, added-directory skills load into the session (workspace
does *not* need its own copy of member-repo skills, only its own spine
skills); if the slash command is unrecognized, they don't (matches hooks'
behavior, and `/workspace`'s design already assumed this — good, but
currently unconfirmed).

### §0.5 — Hook JSON field names / install mechanism. **Fully confirmed by direct reading, no live-fire needed.**

- PreToolUse hooks receive JSON on stdin: `.tool_name`, `.tool_input.file_path`
  (Edit/Write), `.tool_input.command` (Bash). Deny is `exit 2` with a
  human-readable message on stderr (the convention all three existing hooks
  use); `dep-gate`'s "ask" path instead emits
  `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"ask","permissionDecisionReason":"..."}}`
  on stdout and exits 0. Both conventions are real, both already fire
  correctly in production (horizon, bgr) — read directly from
  `core/hooks/phase-gate`, `path-escalate`, `dep-gate`, not inferred.
- Install mechanism (`core/skills/bootstrap/SKILL.md` §5,
  `core/skills/adopt/SKILL.md` §5, confirmed live in `~/horizon/.claude`
  and `~/bgr/.claude` via `ls -la`): per-entry symlinks
  `.claude/skills/<name>`, `.claude/agents/<name>.md`,
  `.claude/rules/<name>.md` into the spine checkout by absolute path, plus
  one whole-directory symlink `.claude/hooks -> <spine>/core/hooks`. A
  **committed** `.claude/settings.json` (not `.local.json`) wires all three
  hooks with matcher `Edit|Write|Bash` under a single `PreToolUse` array
  (horizon's has two separate hook-group entries with the same matcher —
  bgr's has all three in one group; both shapes work, Claude Code doesn't
  care how hooks are grouped as long as the matcher's right).
  `/workspace` needs to replicate exactly this symlink set at the
  workspace root (it's a spine-consuming session like any other) *and*
  additionally populate `permissions.additionalDirectories` with each
  member repo's path — confirmed compatible by test (c) above.

## 2. Contradictions between the build prompt and spine's actual state, reconciled

**None material.** Everything the build prompt asserts about existing
spine mechanics (task classes, the latitude table, the three hooks, the
13-capability adapter contract, `check-stale`, `verdict-filter`,
`docs/decisions/` existing and being distilled from deviations,
`ADAPTER-CONTRACT.md`'s conventions) matches direct reading of `core/` and
`docs/tradeoffs.md` exactly. Two things worth flagging as *manifest
corrections*, not contradictions:

1. **`core/templates/decision.md` already exists** — it is not new. The
   build prompt's own §1 anticipated this correctly ("the same
   `docs/decisions/` store that spine already distills them into after
   deviations... One store, two entry paths"), but §3's deliverable
   manifest lists it as a plain new file. Phase B must **extend** it
   (add `scope` globs, `status`, `supersedes`/`superseded-by` fields to
   the existing Context/Decision/Consequences shape), not create it.
   Concretely, its current docstring states an invariant Extension A
   will deliberately break: *"Distilled by /ship from a resolved deviation
   or an escalation — never written speculatively."* Phase B's edit to
   this file must update that docstring to describe both entry paths
   (`/design`, pre-code; `/ship` distillation, post-code) — leaving the
   old sentence in place while adding a second writer would misdescribe
   the file to the next reader.
2. **`core/skills/ship/SKILL.md` §2 already implements the post-code
   distillation path** (reads `deviations.md`, writes
   `docs/decisions/<date>-<slug>.md` for resolutions "a researcher on a
   later task would actually want to find," explicitly skips low-value
   ones). Extension A's lifecycle work is additive to this — `/ship` needs
   one new step (record implementing paths onto any *cited* adopted
   decision, per the build prompt's "implemented" transition), not a
   rewrite of §2's existing distillation logic.

No other daylight found. The build prompt's description of `floor`,
`ADAPTER-CONTRACT.md`, and the install mechanism all check out exactly
against direct reading (see §3's `floor` note below for one real
*design* consequence, not a contradiction).

## 3. A finding beyond what the build prompt anticipated: the Auto Mode wall is broader than `docs/tradeoffs.md` currently states

`docs/tradeoffs.md` (lines 199–222, "Phase E: made the degradation loud")
asserts the wall is *"confirmed specific to headless/unattended sessions,
not the ordinary case"* — verified there by running `ledger`/`check-stale`/
etc. from a normal attended session with no `-p` flag. This phase found a
narrower but real counter-instance: **an ordinary, attended top-level
session's own Bash tool cannot invoke the `claude` binary itself at all**
(`claude -p ...`, even bare `claude --version`), regardless of
`--dangerously-skip-permissions` or explicit `permissions.allow` entries.
This is not the same claim tradeoffs.md's Phase E section verified (that
section is about *spine scripts* being reachable from an attended session
driving its own tools directly) — it's specifically about *nesting a
`claude` process inside another `claude` session's Bash tool*, which
tradeoffs.md never tested because Phase B/D/E always launched their `-p`
subprocesses from a plain terminal, not from within another live session.

**Practical consequence for this build:** every subsequent phase that
needs to fire-test a hook or skill via a nested `claude -p` subprocess
(Phase B's `check-stale`/`verdict-filter` demonstrations, Phase C's
`/design` and adversary-mandate testing, Phase D's workspace/hook-routing
tests) will hit this same wall if attempted directly via this session's
Bash tool. **Workaround, now validated twice (§0.1 tests a/b/c):** hand the
exact command to the engineer to run via Claude Code's `!` passthrough,
which executes in their own shell outside this session's classifier scope,
then read the resulting file/state changes directly (Read/Bash `cat` are
unaffected — only invoking `claude` itself is blocked). This is slower
(each fire-test is a round trip) but fully real — no test in §1 above was
simulated or asserted without direct evidence.

**Recorded for Phase E's tradeoffs.md extension**, not fixed now: this
narrows what "the wall is specific to headless sessions" means — it should
be restated as "specific to headless sessions *or* to nested `claude`
invocation from within any session, headless or not." Whether this
extends to *all* machines with Auto Mode on, or is specific to some
interaction between nested sessions and the classifier, is outside what
this build can see into (same disclaimer the original finding already
carries).

## 4. Build plan for Phase B–E

Confirmed against direct reading of `core/skills/task/SKILL.md`,
`core/skills/ship/SKILL.md`, `core/skills/verify/SKILL.md`,
`core/skills/bootstrap/SKILL.md`, `core/skills/adopt/SKILL.md`,
`core/scripts/{check-stale,verdict-filter,floor}`,
`core/hooks/{phase-gate,path-escalate,dep-gate}`,
`core/templates/{decision,deviations,CLAUDE}.md`, `core/ADAPTER-CONTRACT.md`.

### Phase B (Extension A's deterministic layer) — concrete integration points found

- **`check-stale`'s header grammar is a hand-rolled `awk` state machine**
  (parses `<!-- spine:research ... sha: ... files:\n  - a\n  - b\n -->`).
  The new `grounding-decisions:` branch must extend this exact grammar —
  likely `grounding-decisions:\n  - D-3@<hash>\n  - D-7@<hash>` — parsed the
  same two-state way (`f=1` on the header line, `/^  - /` rows, exit on
  dedent). Then, per decision ID, resolve `docs/decisions/D-<seq>-*.md`,
  recompute its content hash (open question §5.1's whole-file-vs-body-only
  choice determines exactly what gets hashed), and compare. A second new
  failure mode beyond hash mismatch: **status is `superseded`** — that
  must quarantine the same way, per the build prompt's explicit "hash
  mismatch *or* status superseded" branch. The existing quarantine-banner
  write logic (in-place, idempotent, checks for the banner before
  re-inserting) generalizes without change — only the drift-detection
  predicate grows a second clause.
- **`verdict-filter`'s `valid_evidence`/`valid_verdict` jq functions** need
  a third `evidence.kind == "decision"` branch requiring non-empty
  `decision_id` and `quote`, plus (this is the part the build prompt calls
  out as the actual mechanical check, not just shape validation) the ID
  must resolve to a real `docs/decisions/<id>-*.md` file and the quote must
  appear verbatim in it — `jq` alone can validate presence/non-emptiness
  of fields; resolving-and-grepping the cited file is a shell-level check
  around the existing jq pipeline, not something jq itself can do. Needs a
  small refactor: pull `kept`/`dropped` partitioning into two passes (jq
  shape check, then a bash loop doing the file-resolution check on
  survivors of the first pass) rather than one jq expression.
- **`design-gate`** (new script) has no existing analog to extend — it's
  the one genuinely new script this phase writes. Four checks, all
  mechanical per the build prompt: milestone 0 defined + capability
  targets, every capability in `.spine/capabilities.json` has a planned
  status, every foundational category decided-or-deferred-with-trigger,
  adopted-decision count ≤ cap. All four are greppable/jq-able against
  files `/design` itself will have just written — no new file format
  needed beyond what §2 of the build prompt already specifies for
  `docs/decisions/DEFERRED.md` and `work/<milestone-id>/milestone.md`.
- **Ship-side decision lifecycle**: `core/skills/ship/SKILL.md` needs one
  new step between existing §2 (distill) and §3 (briefing) — for each
  decision *cited* in the shipped plan (plan template would need a new
  optional "Grounds on decisions: D-3, D-7" line, machine-parsed the same
  way `## Predicted touch` already is for `conformance`), append the
  implementing paths (from the plan's own predicted-touch list, now
  confirmed real by the diff) onto that decision record's `Consequences`
  section or a new `## Implementing paths` section, and flip its status
  line to `implemented` if it was `adopted`. This is the "supersession is
  append-only, status-line edits don't change the grounding hash" decision
  (open question §5.1) actually landing somewhere concrete: **whatever the
  hash function is, it must exclude the status line by construction**, or
  this exact ship-time status flip breaks every research.md that already
  cites this decision. Recommend (not yet decided, flag for Phase B):
  hash everything in the file *except* a single well-known `Status:` line,
  matched the same disciplined way `deviations.md`'s `- Status:` line
  already is (`grep`-exact, no punctuation variance).
- Each of the above needs a **real invocation** per the build prompt's own
  Phase B requirement — plan to demonstrate: a hand-authored decision
  record in `adopted` status, a research.md citing it via the new header
  field, `check-stale` passing; then edit the decision's `Decision` body
  (not its status line) and re-run `check-stale`, showing it now flags
  stale; then supersede it (new record, old record's status flipped to
  `superseded`) and show the same research.md now flags via the *other*
  branch (status, not hash). Two real invocations covering both new
  quarantine paths, not one.

### Phase C (`/design`, milestones, adversary design-mode) — scope note

`core/agents/falsifier.md` and `core/agents/security.md` were not read in
full this phase (deferred — Phase C's own job). Skimmed enough via
`ADAPTER-CONTRACT.md §5` to confirm the verdict JSON schema both already
produce; the design-mode mandate sections the build prompt describes are
additive sections in those same files, same output schema, so no contract
change needed beyond `verdict-filter`'s new `decision:` evidence kind
(Phase B). Flag for Phase C specifically: `core/skills/verify/SKILL.md`
§3's adversary-invocation pattern (fresh subagent, plan + diff + artifact
paths only, no session summary) is the pattern design-mode invocation
should mirror — design mode has no diff, so its delegation message is
charter + decision records only, per the build prompt; this is a
*narrower* input than the existing pattern, not a structurally different
one.

The greenfield worked example (§7) needs a real project chosen before
Phase C starts — not decided in this phase, deliberately (matches the
build prompt's own instruction not to resolve architecture/scope questions
unilaterally where a choice exists). Suggest raising it as an
`AskUserQuestion` at the start of Phase C rather than picking it here.

### Phase D (workspace, contracts, staged ship) — one open mechanical question found, not in the build prompt's own open-questions list

`core/scripts/floor` (read in full this phase) dispatches over a **fixed,
hardcoded sequence** of capability names (`typecheck lint`, then
`test secret-scan`, then `dep-diff`, `clone-scan`, `callers`, then
class-conditional `mutate`/`smoke-*`) — there is no generic "run capability
N if some condition" mechanism; each stage is its own `run_*` call written
directly into the script body. Adding `contract-check` as capability #14
per the build prompt is straightforward for capability *declaration*
(`.spine/capabilities.json`, `adapter-conformance`) but **`floor` itself
needs a new conditional stage**, and unlike `mutate`/`smoke-*` (gated on
task *class*, which `floor` already receives as an argument), `contract-check`
must be gated on **whether this repo is in this task's contract blast
radius** — information `floor` does not currently receive at all. This
needs either a new `floor` flag (e.g. `--contracts <name,name>`, populated
by the workspace-level orchestrator from `contract-touch`'s output before
calling `verify` per-repo) or `verify`/`ship`'s aggregation logic invoking
`contract-check` directly, outside `floor`'s own dispatch loop, treating it
more like `conformance` (build prompt §2.5 Layer 4 — never blocks via
`floor`, but unlike `conformance` this one **must** gate, per "the floor
for affected repos is `contract-check` only"). Recommend the latter
(`verify`'s aggregation calls `contract-check` directly per affected repo,
outside `floor`) — simpler, and keeps `floor`'s fixed-sequence design
unchanged for the single-repo case, which is exactly the "zero behavioral
change" requirement in the build prompt's own preamble. Decide for real in
Phase D, this is a recommendation, not a decision.

Also for Phase D: primitive §0.4 (skill/agent resolution across
`additionalDirectories`) must be resolved empirically (test plan above,
§1) *before* `/workspace`'s SKILL.md is finalized — it changes whether
`/workspace` needs to symlink member-repo skills into itself or not.

### Phase E (tradeoffs, self-red-team, v2 shelf)

No new findings beyond what's already flagged above (§3's Auto Mode
extension) — the rest proceeds per the build prompt's own §5/§6/§8
structure once B–D exist to write honestly about.

## 5. What the next session needs that isn't obvious from the files

- **The nested-`claude`-invocation Auto Mode wall (§3) will recur every
  phase that fire-tests a hook or skill.** Budget for it: each live-fire
  test is a round trip through the engineer's `!` passthrough, not a
  single Bash tool call. Write test commands as short scripts on disk
  (avoids terminal line-wrap mangling long inline commands pasted after
  `!` — hit this directly in §1's testing, cost three extra round trips
  before switching to script files) and hand the engineer a one-line
  `bash /path/to/script.sh` to paste instead of a long inline command.
- **`core/agents/falsifier.md` and `core/agents/security.md` were not read
  in full this phase** — Phase C needs to actually read them before writing
  the design-mode mandate sections, this phase only confirmed the verdict
  schema they already produce via `ADAPTER-CONTRACT.md §5`.
- **`core/skills/costs/SKILL.md`, `core/scripts/{ledger,q,adapter-conformance,conformance}`, `core/rules/migrations.md`, and both remaining templates
  (`plan.md`, `map.md`, `briefing.md`, `research.md`) were not read this
  phase.** Phase B needs `research.md`'s template for the
  `grounding-decisions:` header addition; Phase D needs `ledger`'s
  `aggregate` shape for the workspace-level-ledger open question (build
  prompt §5.6) and `q`/`costs` for how `/costs` would report workspace
  aggregation.
- **No core file was modified this phase.** `git status` in `~/spine`
  should show zero changes from this session — confirmed before writing
  this handoff.
- Scratch primitive-test fixtures are session-scratch (`/private/tmp/...`)
  and will not survive past this session — the test methodology (§1) is
  fully reproducible from this write-up if Phase D needs to re-run or
  extend it (e.g. for §0.4's skill-resolution test).
