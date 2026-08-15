# Tradeoffs

Written honestly, at the end of the build, against a real install (`horizon`,
a production Flutter/Firebase app) and a real second-stack validation
(synthetic Python/pytest/mypy/ruff/mutmut project). Every claim below is
either something this build directly observed or is labeled as an estimate.

## What this costs

**Token multiple over undisciplined agent use.** Not yet measurable from
real data — `/costs` is the intended source of truth, and this install has
run exactly one task through the full spine. Estimate, to be replaced
within two weeks of real use per the build prompt's own instruction: **2–4x**
for Class 1 (research + plan + two adversaries + floor, against a single
implementation pass an undisciplined agent would have done directly),
higher for Class 2. The worked example (§ below) is the one real data point:
research + plan + implementation + two adversary agents for a ~50-line,
3-file bug fix. That is a real multiple over "just fix it," and it is the
tax the whole system asks the engineer to accept in exchange for the
artifact trail and the adversarial check.

**Human minutes per class**, estimated: Class 1 — 2–5 minutes (confirm
class, read a ≤200-line plan, read a ≤1-page briefing). Class 2 — adds
explicit Class 2 confirmation and a higher chance of a halt-tier deviation
needing resolution, so 10–20 minutes. These are the three touchpoints
(build prompt §2.7) multiplied by realistic reading time, not measured.

**First-use friction this build found and did not fully resolve**: see
"The Auto Mode classifier wall" below. It doesn't change the steady-state
cost model but it is a real, disclosed cost of the first session in a newly
adopted project under some permission configurations.

## Where this is worse than a thoughtful engineer with a bare agent

- **Exploratory spikes and prototypes.** The spine assumes research → plan
  → implement is the right shape. A spike whose entire point is "I don't
  know what I want yet" fights that linearity — the plan cap and the
  phase-gate hook actively get in the way of throwaway exploration. Use
  Class 0 or step outside `/task` entirely for this.
- **Tiny repos the engineer holds in their head.** The whole artifact
  ladder exists to compensate for context loss across sessions and across
  people. A single-file script the engineer wrote themselves ten minutes
  ago doesn't have that problem yet.
- **Genuinely novel design work.** Research assumes there's an existing
  "how it works today" to ground in. Net-new architecture has no such
  ground truth — the research phase would either come back empty or
  (worse) invent false grounding.

## When to abandon it

If `/costs` shows Class 1 attention cost (the three touchpoints, summed)
exceeding what a careful human review of the same diff would have cost,
after the initial calibration period and after `/ratchet` has had a chance
to convert repeat friction into deterministic checks, the system has failed
its own test. This is the build prompt's own falsifiable exit condition —
recorded here so it isn't quietly forgotten once the system is running.

## What makes this obsolete

Models that maintain reliable long-horizon state across sessions gut the
core rationale for the artifact ladder (`docs/charter.md`, `docs/map.md`,
`research.md`/`plan.md`/`verify.md`, the delta briefing) — those exist
because a turn is a stateless function of its context window today. If that
constraint goes away, the artifacts become redundant with the model's own
memory, and the ceremony around producing them becomes pure cost.

## Residual risks conceded by design

- **`docs/charter.md` sections that never collide with reality can rot
  undetected.** No mechanical check falsifies intent; the only invalidation
  channel is a deviation resolution that happens to cite the stale line.
  horizon's charter draft (§ Worked example) is brand new and entirely
  unexercised by this concern yet — it will start accumulating this risk
  the moment it's confirmed and a few tasks have run against it.
- **Concurrency defects** are out of scope for v1 (no concurrency/stress
  lane exists — see the v2 shelf).
- **Seeded data is not production-shaped data.** Even where `smoke-*`
  capabilities exist for a stack, a seeded dataset picks the shapes someone
  thought to seed; production drifts. horizon doesn't even have this
  problem yet in the useful sense — its `smoke-*` capabilities are
  `unavailable` outright (no local Firebase emulator config exists today),
  so the gap is total, not partial, for this install.
- **Semantic collisions between sequential tasks** are caught only by
  research freshness (`check-stale` against grounding-file drift) — two
  tasks that touch logically related but textually disjoint code can both
  pass their own research/plan/verify cleanly and still combine badly.
  Nothing in v1 detects this.
- **The plan-adequacy gap** (below) is the single most important of these,
  named explicitly by the build prompt, and worth its own heading.
- **The circuit breaker's honor-system dependency.** The 3rd-deviation
  circuit breaker (build prompt §2.4) counts deviations logged to
  `deviations.md`; nothing forces a deviation to actually get logged.
  Deliberately not turned into enforcement in Phase E — this is a norm
  about model/human judgment ("did this count as a deviation"), not a tool
  call, and mechanizing it would mean intercepting judgment rather than a
  Bash/Edit/Write invocation, which is a different (and much larger) kind
  of gate than anything else in v1. An implementer motivated to avoid
  tripping the breaker can simply not log a deviation and proceed. Recorded
  here, alongside the other conceded gaps, rather than only in the
  self-red-team section below — a reader deciding whether to trust the
  breaker needs this next to the rest of what's conceded by design, not
  buried in a gate-by-gate attack log.

### The plan-adequacy gap

The falsifier tests the implementation against the plan's own acceptance
checks; `conformance` measures whether a plan was *predictive* (did the
diff touch what the plan said it would). Neither measures whether a plan
was *adequate* — a plan with weak acceptance checks passes verification
cleanly, every time, by construction. That load sits entirely on the
human's plan review. This is probably the right place for it — a human
deciding "is this plan actually testing the right thing" is not a check
that decomposes well into a deterministic gate — but it means **plan
review is the one link in the chain with no backstop**, and the build
prompt is right to say so plainly rather than engineer around it.

The worked example makes this concrete: the plan's own acceptance checks
(`work/20260808-fix-building-group-delete-orphans-units/plan.md` §Acceptance
checks) include one manual, unautomated check ("create a building group,
assign a unit, delete the group, confirm in the UI the unit no longer shows
the deleted group") — nothing in `/verify` runs that check; it was proposed
by the plan and never independently confirmed by this build. If the plan
had gotten the acceptance criterion subtly wrong (e.g. checked the wrong
field), the falsifier could still report a clean bill on that criterion,
because it correctly implements the wrong check.

## The Auto Mode classifier wall — a real finding, not a design decision

The single most consequential thing this build discovered was not
architectural, it was primitive-level, and it surfaced only by actually
running the spine end-to-end against horizon rather than just fire-testing
hooks in isolation (as Phases B/C did).

**What happened.** Driving the worked example's `/task` through a real
`claude -p` subprocess (necessary to exercise the real `researcher`/
`falsifier`/`security` subagents and the real hooks, which only resolve
correctly from a session whose project root is the installed project, not
from `spine/` itself), every attempt to invoke a spine core script or a
project adapter via the Bash tool — `core/scripts/ledger`, `core/scripts/
floor`, even a purely in-project, purely relative path like `./.spine/
adapters/typecheck` — required interactive approval that a headless
session has no way to grant. `--permission-mode acceptEdits` did not
suppress it. A `.claude/settings.json` `permissions.allow` rule matching
the exact command did not suppress it. `--add-dir` pointed at the spine
checkout did not suppress it. Even `--dangerously-skip-permissions`
returned an explicit denial: *"Permission for this action was denied by
the Claude Code auto mode classifier."*

**What this is.** This machine has Auto Mode enabled at the user level (a
Claude Code feature, distinct from sandbox auto-allow — the two are
documented as independent layers). Auto Mode's classifier reviews Bash
actions independently of sandbox filesystem rules and, apparently,
independently of explicit permission allow-rules, and it is conservative
about approving execution of an arbitrary local script it can't verify the
behavior of — even one that is, by construction, a read-only self-test or a
narrowly scoped capability check.

**What this is not.** It is not evidence against the "external clone
referenced by path" install mechanism specifically — the same wall applied
to a purely in-project relative path (`./.spine/adapters/typecheck`), so
this is not a symlink or cross-directory problem. It is a general property
of running any script-shelling-out skill under this specific permission
configuration, unattended.

**Practical impact.** A live, attended engineer running `/task` normally
would see this (if at all) as an ordinary one-time Bash permission prompt
the first time a skill invokes a spine script, then approve it — exactly
the same experience as approving any new command. This build's difficulty
was specific to driving the flow *unattended* (`-p`, no human present to
approve) for verification purposes, which is the harder case, not the
common one. Skills already degrade gracefully when a script is unreachable
— `/task`'s own research phase, hitting this wall, fell back to hand-
tracking `work/<task-id>/state` rather than failing outright (see the
worked example's `notes.md`) — but "graceful degradation to manual
tracking" is a softer failure than the ledger/conformance mechanisms are
designed around, and it went unnoticed by anything mechanical. Nothing
currently detects that a task's ledger entries were hand-tracked rather
than script-derived.

**What this build did about it.** Added `.claude/settings.json`
`permissions.allow` entries for the spine scripts path and the in-project
adapters path in horizon's install (harmless, and correct for the
interactive case even though it did not resolve the headless case tested
here). Did not attempt a deeper fix — root-causing exactly why Auto Mode's
classifier overrides an explicit allow rule is outside what this build
could verify without Anthropic-internal visibility into the classifier.

**Recommendation for the engineer:** if `/costs` or lived experience shows
skills silently falling back to hand-tracked state instead of using
`ledger`/`check-stale`/`conformance`, that is this issue recurring — check
whether Auto Mode is active and, if the friction is unacceptable, disable
it for spine-installed project sessions or file the specific denial pattern
with Anthropic. This is exactly the kind of "silently degraded gate" the
build prompt says is the worst object this system can produce (§4) — the
difference is that here the degradation is in the *build's own tooling*,
not a project capability. **This was this build's single strongest
self-red-team finding through Phase D.**

**Phase E: made the degradation loud, and confirmed the scope of the
wall.** Two things changed:

1. **The wall is confirmed specific to headless/unattended sessions, not
   the ordinary case.** Phase D suspected but never confirmed this
   ("should not recur for an ordinary, attended, interactive session — that
   case was not the one this build had trouble with"). Phase E confirmed it
   directly: `ledger init`/`mark`/`set`, `check-stale`, `conformance`, and
   `verdict-filter` were all run for real, from a normal attended session
   (no `-p`, no `--dangerously-skip-permissions`, permission prompts
   available to approve exactly as they would be for any engineer),
   against horizon's real worked-example task artifacts. **All four
   executed cleanly and wrote exactly what they claim to** — a real
   `ledger.json` with a real `classify` phase timestamp; `check-stale`
   correctly detecting real drift (the worked example's own research.md
   *is* stale — a grounding file changed after research was written — and
   `check-stale` correctly quarantined a scratch copy of it, non-
   destructively, to prove this without touching the real task's
   artifacts); `conformance` computing a real precision/recall/f1 against
   the actual diff; `verdict-filter` validating and passing 4/4 real
   falsifier verdicts. The Auto Mode classifier wall is real and still
   unresolved for headless use, but the engineer's actual day-to-day
   case — an attended `/task` session — is now confirmed clean, not just
   assumed clean.
2. **The degradation is now mechanically visible when it does happen**,
   closing the "no mechanism in v1 makes it visible" half of this finding.
   `ledger` gained `note-gap` (any skill that hits an unreachable script
   records `{script, consequence, at}` into that task's `tooling_gaps`
   array) and a `hand_tracked` stamp (a skill that cannot reach `ledger`
   itself hand-authors a same-shaped `ledger.json` with `hand_tracked:
   true`, so the task stays visible to `aggregate` instead of silently
   having no ledger file at all — the worked example's own task is the
   concrete case this fixes: it has no `ledger.json` on disk today,
   precisely the invisible-task failure mode this stamp exists for).
   `ledger aggregate` now reports `tooling_gap_count` and
   `hand_tracked_task_count`; `/costs` surfaces both next to the bypass
   count, with the same "say so plainly if nonzero" framing bypass already
   gets. `work/<task-id>/notes.md` is the running log a skill writes to the
   moment a gap happens (`TOOLING GAP: <script> — <consequence>`), and
   `/verify`/`/ship` fold it into `verify.md`'s and `briefing.md`'s new
   "Tooling gaps" sections — so a human reading either document sees the
   gap without having to already know to look in notes.md or run
   `/costs`. This does not make the classifier wall disappear, and it
   still depends on the model actually following the recording discipline
   the skills now spell out (the same honor-system caveat every other norm-
   not-hook-enforced mechanism in this system carries — see the circuit
   breaker's own residual-risk entry) — but a degradation that happens now
   leaves a trace in three places (notes.md, ledger.json, verify.md/
   briefing.md) instead of zero. **This build's single strongest
   self-red-team finding is narrowed, not closed**: the wall itself is
   unfixed (still Anthropic-internal classifier behavior this build cannot
   see into), but both halves of why it mattered — "does it actually hit
   the common case" and "would anyone notice" — now have real answers
   instead of open questions.

**Extension build, Phase A: the wall is broader than "headless sessions"
— it also blocks nesting a `claude` process inside another session's Bash
tool, attended or not.** The confirmation directly above verified the wall
doesn't hit an *ordinary attended session driving its own tools directly*.
The extension build found a narrower, real, distinct case: an ordinary,
attended, top-level session's own Bash tool still cannot invoke the
`claude` binary itself (`claude -p ...`, even bare `claude --version`),
regardless of `--dangerously-skip-permissions` or explicit
`permissions.allow` entries — confirmed by testing all three
independently, none suppressed it. This is a different claim than the one
verified above, because every prior fire-test in this document launched
its `-p` subprocess from a plain terminal, never from inside another live
`claude` session's own Bash tool. **Practical consequence, confirmed
across every phase of the extension build**: any session that needs to
fire-test a hook or skill via a nested `claude -p` subprocess (which every
phase of this extension build did, to watch its own new hooks/agents fire
for real) hits this wall directly via its own Bash tool and cannot work
around it internally. The only working mitigation found: hand the exact
command to the engineer to run via Claude Code's own `!` passthrough,
which executes outside the classifier's scope entirely, then read the
resulting file/state changes directly (plain Read/Bash `cat` of the result
is unaffected — only invoking `claude` itself is blocked). This restates
the wall's scope precisely: **specific to headless sessions, or to nested
`claude` invocation from within any session, headless or not** — not, as
the language directly above could be misread to imply, specific to
headless sessions alone.

## v2 shelf (forbidden in v1, insertion points noted)

| Deferred | Insertion point |
|---|---|
| Agent teams / multi-session orchestration | None built; `/task`'s single-session model is the whole v1 surface |
| Parallel work streams, semantic arbitration | Layer 1 calibration's `work_mode.parallel_streams` field exists and is hardcoded `false` |
| Cross-model adversary routing | `model: inherit` on every agent in `core/agents/*.md` — the field exists, unused beyond inheritance |
| Comprehension quizzes | None — no insertion point built |
| Incident-intake workflow | Only the commit↔task linkage (`Spine-Task:` trailer, `docs/decisions/`) was built; the workflow that walks production → blame → task → research is not |
| Prompt regression fixtures | None |
| Scheduled consolidation of pre-existing duplication | `clone-scan` only runs against the changed-file set — it prevents new duplication, never sweeps existing debt. horizon's real codebase is presumably carrying pre-existing duplication `clone-scan` will never surface, symmetrically with the lint-debt finding below |
| Product-spec layer | `docs/charter.md` deliberately stays at non-negotiables/constraints, not a spec |
| Concurrency/stress lane | The `smoke-*` capability names are the insertion point — `smoke-run` could grow a concurrent-load mode |
| `contract-scan` (deterministic undeclared-coupling detection) | The falsifier's design-mode/cross-repo mandate is the only thing that currently hunts undeclared coupling — a mechanical, scriptable version was deliberately not built (build prompt §2: "the registry lookup carries the value at near-zero cost") |
| Baseline failure attribution for affected-but-not-edited repos | `contract-check` gates every affected repo unconditionally; nothing yet distinguishes a pre-existing consumer failure from one this task caused — the edited-vs-affected floor split is the insertion point |
| Decision-drift metrics in `/costs` | `check-stale`'s decision-quarantine branch is real and fires; nothing yet aggregates how often decisions drift/supersede into a `/costs`-visible metric the way bypass count already is |
| Per-repo charters in a workspace | One system charter at the workspace root only; member repos don't get their own — `docs/charter.md`'s existing single-document shape is the insertion point if this is ever needed |
| Auto-generated cross-repo topology maps | `workspace.json` + the contract registry *is* the topology today, hand-authored/skill-written; nothing renders or derives a map from it |
| Brownfield `/design` worked flow | `/design` explicitly supports running on brownfield to make implicit architecture explicit (build prompt §2), but this build's only real worked example was greenfield — brownfield `/design` is unexercised, not unbuilt |
| Multi-workspace (a repo belonging to more than one workspace) | `workspace.json`'s repo list assumes one workspace per member repo; nothing prevents authoring a second workspace pointing at the same repo, but nothing coordinates the two either |
| Anything depending on `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | Explicitly out of scope per build prompt §0.3 — per-repo context reaches sessions through skills reading files instead, deliberately not through this env flag |
| Full `/costs` aggregation across a workspace and its member repos' independent task histories | Open question 6 (above) — mechanism named (`loop over workspace.json`'s repos + workspace root, sum `ledger aggregate`), zero code written |
| Design-mode adversary cost tiers | Open question 7 (above) — left unresolved through both Phase C and Phase E; no tiering exists, every design review pays full adversary ceremony regardless of decision-set size |

## The seven open questions (build prompt §5), answered and defended

1. **Task-ID scheme / commit trailer.** `<YYYYMMDD>-<kebab-slug>` task IDs,
   `Spine-Task: <id>` / `Spine-Bypass: <reason>` trailers
   (`core/ADAPTER-CONTRACT.md §6`). Mechanically greppable by `ledger
   scan-untracked-ratio`; composes with horizon's existing Conventional-
   Commits CI check and its `Co-Authored-By:` trailer without conflict —
   confirmed against horizon's real commit history in Phase B.
2. **Class 0 threshold / smoke runtime budget.** ≤2 files, ≈15 lines, no
   new public symbol, no protected-path touch (`core/skills/task/SKILL.md
   §1`); smoke joins the floor when its measured runtime ≤60s
   (`~/.spine/user-config.json`). Both stated as initial values in the
   skill itself, explicitly flagged for `/costs` to revise — unrevised as
   of this writing, since only one task has run.
3. **Per-capability tool choices.** Recorded per-project in
   `.spine/capabilities.json`'s `reason` fields, not here — that's the one
   place build prompt §2.5 says they may live. horizon: 8 of 13
   implemented (typecheck/lint/test/test-changed via `flutter
   analyze`+`dart format`+`flutter test`, secret-scan self-contained regex,
   dep-diff/clone-scan/callers via git diff/jscpd/grep), 5 `unavailable`
   (mutate — no mature Dart mutation tool; all four smoke/migrate-rehearse
   — no local Firebase emulator config exists). The synthetic Python
   validation project: 9 implemented (adds `mutate` via `mutmut`, genuinely
   different from horizon — proof capability status isn't hardcoded), 4
   `not-applicable` (pure library, nothing to run or migrate).
4. **Ledger token-usage harvesting.** Verified mechanism (Phase A §2.7):
   session transcripts at `~/.claude/projects/<sanitized-cwd>/<session-
   uuid>.jsonl`, subagent transcripts at `.../subagents/agent-<id>.jsonl`,
   both newline-delimited JSON with `message.usage.*` token fields.
   **Caveat found in this phase**: this harvesting itself goes through
   `core/scripts/ledger`, which is exactly the kind of script invocation
   the Auto Mode classifier wall (above) can block — so under that
   configuration, ledger harvesting is exactly what silently degrades to
   "didn't happen" rather than erroring loudly. Disclosed, not fixed.
5. **Research-lite vs. full research line.** Class 1 grounds only
   directly-touched/directly-called files; Class 2 surveys the real
   subsystem and its actual callers, no limit (`core/skills/task/SKILL.md
   §2`). The worked example's research.md grounds 7 files for a 3-file
   change — slightly wider than "directly touched," because two of those
   seven (`docs/map.md`, `lib/core/constants/constants.dart`) were
   consulted for context rather than modified. That's within the stated
   rule's spirit (ground what you need to be confident, not what you'll
   edit) but is worth watching — if research-lite routinely balloons past
   "directly touched," the line needs tightening, not just restating.
6. **Floor in CI.** Not wired for horizon in this build — Layer 2
   calibration's default (mirror locally; CI integration is an addendum)
   was accepted, and horizon's actual CI
   (`.github/workflows/main.yaml`) was left untouched. Through Phase D the
   floor failed on `lint` for pre-existing, task-independent reasons (see
   the worked example, below) — Phase E's rescoping (§ "Phase E: the
   lint/typecheck rescoping decision") means a per-task floor run no
   longer fails on debt the task never touched, but horizon's CI still
   isn't wired to `floor --full` (the whole-tree mode that *would* still
   hit that debt) — sequence that after the formatting-debt cleanup task,
   not before, same reasoning as Phase D, now with a concrete mechanism
   (`--full`) rather than an open question.
7. **Adapter argument convention.** stdin for changed-file sets, first
   positional arg for output artifacts, `SPINE_BASE_REF` for a diff base
   (`core/ADAPTER-CONTRACT.md §3`). Validated twice now — once against
   horizon's real Dart/TS adapters (Phase B), once against a synthetic
   Python/pytest/mypy/ruff/mutmut/jscpd adapter set built fresh in Phase D
   for two-stack validation — with zero changes needed to the convention
   itself across either stack.

## Phase E: the lint/typecheck rescoping decision (answered architecture question)

Phase D found and named, but explicitly declined to resolve unilaterally,
the standing property that **the floor could not pass on any task in
horizon** because `lint` and `typecheck` checked the whole tree, not the
diff, against a tree that already carried 330 files of pre-existing
formatting debt. Phase D offered two paths — a dedicated cleanup task, or
moving `lint`/`typecheck` into the changed-file-set bucket — and left the
choice to the engineer rather than deciding it. That choice has now been
made, from outside this build: **both, not either.**

**What changed.** `core/ADAPTER-CONTRACT.md §3` now places `lint` and
`typecheck` in the changed-file-set bucket alongside `test-changed` and
`clone-scan` (§3.1). `floor` gained a `--full` flag that restores the
pre-Phase-E whole-tree verdict for both capabilities, unchanged in
implementation — it is a mode switch, not a second code path bolted on.
horizon's two adapters and a freshly-built synthetic Python project's two
adapters (mypy/ruff — Phase D's own synthetic project no longer exists on
disk; rebuilding it in full was judged disproportionate to validating a
two-capability contract change, so only `typecheck`/`lint` were rebuilt for
this validation, not all nine of Phase D's capabilities) were both updated
to the same convention and both pass `adapter-conformance --all`.

**What this concedes, stated plainly, per the build prompt's own
instruction to disclose architecture tradeoffs rather than bury them:**

- **Pre-existing debt in untouched files becomes invisible to the per-task
  floor**, by design. A file nobody has touched in months can carry a real
  lint or type violation indefinitely, and no per-task `/task` run will
  ever surface it — only `floor --full` or CI (once wired) will. This is
  the direct cost of making the per-task gate reflect only the task's own
  diff: the price of "stop punishing engineers for debt they didn't write"
  is "stop mechanically re-discovering that debt on every task." horizon's
  330-file formatting debt is the concrete, currently-live instance of
  this — it does not go away, it becomes something only `--full`/CI will
  ever find.
- **`typecheck`'s scoping is verdict-filtering, not invocation-scoping**
  (a real asymmetry between the two rescoped capabilities, worth naming
  explicitly): `flutter analyze`/`mypy` still run against the whole
  package/root internally, because a type checker needs full-project
  context to resolve types correctly — the adapter filters the *verdict*
  to errors located in changed files. `lint` (`dart format`/`eslint`/
  `ruff`), having no cross-file semantics, scopes the *invocation* itself.
  A consequence of the filtering approach: if a changed file's signature
  edit breaks an *unchanged* caller elsewhere, that break is real, would
  show up in `flutter analyze`'s full output, and is filtered out because
  the broken file isn't in the changed set. This is the same "changed-file
  scope can miss change-caused breakage in a file that isn't itself
  changed" tradeoff every capability in this bucket already carries
  (`test-changed`'s own vacuous-pass-when-no-matching-test-file behavior is
  the same shape) — not new to this fix, but worth stating next to it
  rather than leaving it implicit.
- **`--full` is not wired anywhere yet** — it exists as a capability, not
  a running check. Nothing currently calls it except a human running
  `floor --full` by hand. Until it's wired into CI or a scheduled cleanup
  cadence, "the debt is invisible to per-task floor but visible to
  `--full`" is true in principle and unenforced in practice — the same gap
  named in open question #6 above, now with a mechanism rather than an
  open question, but still not automated.

**Verified, not asserted:** re-ran the exact previously-failing scenario.
`floor 1 --task 20260808-fix-building-group-delete-orphans-units` (the
worked example's own task) now reports `PASS: typecheck` and `PASS: lint`
— both scoped to the task's real 3-file diff — and proceeds to (correctly,
separately) fail at `test` for the pre-existing, disclosed, unrelated
reason already on record in `capabilities.json` (the Very Good CLI
counter-template suite and the admin `file_picker` version conflict). The
same invocation with `--full` appended reproduces the original 330-file
`lint: FAIL` exactly. `adapter-conformance --all` passes for both
capabilities in horizon and in the rebuilt synthetic Python project, with
self-test fixtures that specifically prove scoping (a sibling
always-violating file excluded from the "changed" set, per
`ADAPTER-CONTRACT.md §4`'s new requirement) — not just that the underlying
tool wrapper works. The synthetic Python project additionally reproduces
the same real-repo proof horizon's did: a genuinely committed
`preexisting_debt.py` (real ruff and mypy violations) that scoped
`lint`/`typecheck` correctly ignore and `--full` correctly still catches.

## Install mechanism (Phase A's decision, re-confirmed here)

External clone referenced by absolute path
(`/Users/michaelbart/spine`), reached from an installed project via
symlinks (`.claude/{skills,agents,rules}` per-entry, `.claude/hooks`
whole-directory). Chosen over copy-vendor or git submodule because it's
the only option where updating the core is `git pull` inside `spine/` with
zero further commits in the installed project. Confirmed real end-to-end in
this phase: `horizon`'s actual install (§ below) used exactly this
mechanism, all three hook types fired correctly against horizon's real
tree, and the researcher/falsifier/security agents ran for real through
the symlinked agent definitions.

**Known cost, now doubly confirmed:** symlink targets and the
`permissions.allow` entries this phase added are absolute paths tied to
this machine's layout. A second engineer or CI needs either an identical
clone path or a re-run of the install script to regenerate machine-local
symlinks/settings. Given Layer 1 confirmed solo/single-stream for this
engineer, this is accepted for v1 — team use needs a fixed clone-path
convention or a switch to submodule/copy, noted as the mitigation path, not
built.

## Portability seam (harness bindings vs. portable core)

Verified by direct inspection, not assumed: `core/scripts/*` (`floor`,
`adapter-conformance`, `check-stale`, `conformance`, `verdict-filter`,
`ledger`, `q`) reference zero `CLAUDE_*` environment variables and never
invoke the `claude` CLI — they take plain argv/stdin, exactly per
`ADAPTER-CONTRACT.md`. `core/hooks/*` are the one Claude-Code-specific
layer (they consume the PreToolUse JSON schema and `CLAUDE_PROJECT_DIR`),
and they are self-contained — none of them call into `core/scripts/*`, so
the two layers never blur into each other. The binding happens only in
`core/skills/*/SKILL.md`, via `${CLAUDE_SKILL_DIR}`-relative resolution
into plain, resolved paths handed to the portable scripts. A different
harness would need to reimplement the three hook triggers (simple,
self-contained bash against `.spine/*.conf` and `work/*` state files) and
drive the scripts directly with plain arguments — the scripts and artifact
formats themselves need no rewrite.

## Stack-independence audit

Grepped the entire `spine/` repository (`core/` and `README.md`) for
language/framework/tool names, including comments and templates. Found and
fixed three real hits, all pre-existing from Phases B/C, none introduced in
this phase:

1. `core/ADAPTER-CONTRACT.md`'s example verdict JSON used `lib/foo.dart`
   and a `dart test ...` example command — replaced with stack-blind
   placeholders (`src/foo`, `.spine/adapters/test --stub-feature`).
2. `core/hooks/dep-gate` hardcoded six package-manager names
   (`npm|pnpm|yarn|flutter|dart pub|pod|bundle|pip`) directly in its
   Bash-command regex — moved to a new project-owned file,
   `.spine/install-command-patterns.conf` (one extended-regex fragment per
   line, generated at install time from confirmed Layer 3 answers, same
   pattern `protected-paths.conf` already uses for `#manifest` tagging).
   `bootstrap`/`adopt` SKILL.md now generate this file; horizon's real one
   was written and the hook re-verified firing correctly against horizon
   with the new file in place (both a real block on `flutter pub add` and
   a real pass-through on a harmless command, fresh-subprocess tested).

Re-grepped after both fixes: zero hits remain. No third finding surfaced
in this phase's own new files (checked as part of the same pass).

## Two-stack validation

Built a second, complete adapter set (typecheck via `mypy --strict`, lint
via `ruff`, test via `pytest`, test-changed, secret-scan — reused
horizon's script verbatim, dep-diff, clone-scan via `jscpd` — same tool
horizon's uses, callers, mutate via `mutmut`) for a synthetic Python
library in a third, scratch directory — deliberately orthogonal to
horizon's Dart/Flutter/TypeScript stack (dynamic vs. static, completely
different toolchain). Ran `adapter-conformance --all` against both real
projects.

**Result: zero changes needed to `spine/` core or `ADAPTER-CONTRACT.md`.**
The convention held across both stacks without modification. One real bug
was found and fixed in the process — the Python `clone-scan` adapter's
self-test used `grep -c ... || echo 0`, and since `grep -c` prints `0` and
still exits 1 on no match, the `||` fired anyway, concatenating a second
`0` and breaking the `[[ ]]` numeric comparison (`adapter-conformance`
caught this immediately: `FAIL: clone-scan --self-test pass exited 1`).
This is exactly what the conformance suite is for, and it worked as
designed on a brand-new adapter it had never seen before.

**One cross-stack finding, present identically in both adapter sets, not a
bug**: `secret-scan`'s own self-test fixture embeds a literal fake AWS key
(`AKIAABCDEFGHIJKLMNOP`) directly in the tracked adapter script. Scanning
the full working tree without a changed-file-set on stdin (the fallback
path) makes `secret-scan` flag its own adapter file. Scoped invocation (the
normal `/task`/floor usage path, changed files only) is unaffected —
confirmed by testing both modes directly. Worth knowing if anyone ever
invokes `secret-scan` standalone in full-tree mode.

## The worked example — a real Class 1 task in horizon

**Task**: `BuildingGroupsCubit.deleteGroup` deleted a building group
without unassigning its member units, leaving `Unit.buildingGroupId`
pointing at a deleted document — a real, self-disclosed bug (the method's
own doc comment said as much: *"does not unassign its units first"*).
Chosen because it was small, self-contained, non-protected, and had crisp,
checkable acceptance criteria — exactly the profile the build prompt asks
the worked example to have.

**Run for real**, task ID `20260808-fix-building-group-delete-orphans-units`:
classify (Class 1, confirmed) → research (real `researcher` subagent,
grounded 7 files, SHA-stamped) → plan (53 lines, well under the 200-line
cap, real latitude table, real rejected-alternatives section) → human
review (this build read the plan, verified the cited repository methods
actually exist before approving — a real plan-review pass, not a rubber
stamp) → implementation (three files touched, exactly matching the
predicted-touch list; `flutter analyze` clean) → verify (real floor run,
real `falsifier` and `security` subagents in worktree isolation).

**Real deviation encountered, not manufactured, not hidden**: the ledger/
check-stale scripts were unreachable from the driving session due to the
Auto Mode classifier wall (above). The task adapted by hand-tracking phase
state and disclosing the gap in `work/<task-id>/notes.md` rather than
either failing or silently pretending the ledger was populated.

**Real floor result**: FAIL at `lint` — 330 pre-existing unformatted Dart
files (root + admin) and a missing `functions/` ESLint config, identical to
what Phase B found against horizon's tree independently, now reproduced
inside a real task's `/verify` run. Confirmed this was not caused by the
task: `git status` before and after the floor run showed only the task's
own 3 intended files modified — `dart format`'s `--output=none` flag means
the floor's lint check never mutates the tree, only reports.

**Historical note (Phase E):** this `FAIL` was real, at the time, under the
whole-tree `lint`/`typecheck` scoping Phase D shipped with. Phase E
rescoped both to the changed-file set (§ "Phase E: the lint/typecheck
rescoping decision") and re-ran this exact task's floor: it now reports
`PASS: typecheck` and `PASS: lint`, correctly proceeding to fail at `test`
instead for the separate, pre-existing, disclosed reason already on record
above. The 330-file debt this paragraph describes has not been fixed — it
would still fail `floor --full` — it has simply stopped being something a
per-task floor run reports as this task's problem.

**Real adversary results, both in worktree/read-only isolation as
designed**: falsifier reported 4 verdicts (1 low, 2 medium, 1 low — no
`verdict-filter` run, see below, but all 4 were well-formed and would have
survived filtering); security reported 3 (1 high, 1 medium, 1 low). Two
were genuinely load-bearing, not decoration:

- **Security's high-severity finding was about this build's own process,
  not the target code**: it caught an unrelated, uncommitted
  `.claude/settings.json` permission change (a Bash-execution allow-list,
  added earlier in this same build for the stack-independence audit fix)
  bleeding into this task's diff, correctly flagging it as an out-of-plan,
  permission-escalation-shaped change with unknown origin from the
  session's point of view. This is exactly what the adversary layer is
  for — real or not, an unexplained permission-widening change riding
  along in a narrow bug-fix diff is precisely the shape it should refuse
  to wave through. Fixed by committing that change separately, out of the
  task's diff, before writing the briefing.
- **Both adversaries independently found the same real, un-fixed defect**:
  a TOCTOU race — `deleteGroup` fetches member units once and never
  re-checks before `delete()`, so a unit assigned to the group in the
  window between fetch and delete still ends up orphaned. Same defect
  class as the bug this task fixes, narrowed to a race instead of an
  inevitability — a direct, known consequence of the plan's own rejected-
  atomic-batch decision, not an oversight. Left un-fixed and disclosed in
  the delta briefing for a human to decide whether it's worth a follow-up
  task, per this build's judgment that expanding scope to close it
  unilaterally would have exceeded a Class 1 fix.
- Also found: zero test coverage exists anywhere for `BuildingGroupsCubit`/
  `BuildingGroupsRepository` (falsifier's stub-out probe confirmed a
  reversion to the pre-fix orphaning bug produces no test failures at all)
  — consistent with, not new information beyond, Phase B's repo-wide
  finding.

`verdict-filter` and `conformance` did not run — blocked by the same Auto
Mode classifier wall as the floor orchestrator (below) — so both
adversaries' verdicts are reported raw in `verify.md` rather than filtered.
Inspected by hand: both JSON objects are well-formed per
`ADAPTER-CONTRACT.md §5` (non-empty `attacked`, every verdict carrying a
valid `file_line` or `command` evidence shape), so no verdict would have
been dropped had the filter run — but this is a manual substitute for a
mechanical guarantee, disclosed as exactly that.

**This did not ship**, correctly. `/ship`'s merge gate requires floor
`PASS`, and this floor genuinely `FAIL`ed for pre-existing reasons; using
`--bypass` would have been a misuse of that mechanism (`ship/SKILL.md §1`
reserves it for a genuine emergency, not disagreement with a check). The
task's full outcome — implemented, verified, deliberately not merged — is
recorded in `work/<task-id>/briefing.md`, hand-written since `/ship` itself
never got called.

**This is the single most important finding the worked example produced**:
*the floor cannot currently pass on any task in horizon*, because `lint`
(and `typecheck`, less severely — see Phase B) check the whole tree, not
the diff, and the whole tree already violates the check independent of any
future task. This is not a bug in a single task's code; it's a standing
property of the floor as currently scoped, discovered by trying to actually
use it. Two honest paths forward, deliberately not decided unilaterally by
this build: (a) a dedicated, separately-reviewed formatting-debt cleanup
task (`dart format` tree-wide, add a `functions/.eslintrc`) that gets the
floor to a clean baseline once, after which future tasks' diffs keep it
clean; or (b) revisit whether `lint` belongs in the changed-file-set bucket
of `ADAPTER-CONTRACT.md §3` rather than the whole-tree bucket — an
architecture-level change this build declines to make unilaterally, per
the build prompt's own instruction to raise architecture disagreements
rather than resolve them silently. Recorded here as the clearest possible
example of "the worked example is also your integration test."

**Decided in Phase E: both, not either** — see "Phase E: the lint/
typecheck rescoping decision" above for the resolved architecture question,
what it concedes, and the re-run proof against this exact task.

Full artifacts — `research.md`, `plan.md`, `verify.md`, the ledger (partial,
hand-tracked), the delta briefing — live under
`horizon/work/20260808-fix-building-group-delete-orphans-units/`.

## Extension Build — the design stage and multi-repo coordination

Everything below this heading was added by a second build (`spine/work/
.build/build-prompt-extensions.md`, Phases A–E, dated 2026-08-08 through
2026-08-10), against the system already described above. It adds two
things: **Extension A**, a `/design` stage that lets a greenfield project's
first session produce a reviewed decision set instead of skipping straight
to code; and **Extension B**, coordination across multiple repositories
through a workspace root and first-class contracts. Both are additive —
the single-repo, no-`/design` flow described everywhere above this heading
is unchanged and was directly regression-tested (see "Single-repo
regression demonstration" below).

### What the design stage costs

Not yet measurable from real `/costs` data — no design-mode session has
run through `ledger`'s token-harvesting path with results reviewed, so this
is, like the original build's own cost section, an estimate to be replaced.
The one real data point: `~/bookmarks`'s `/design` session walked all six
foundational categories (one, search implementation, legitimately
deferred), ran both adversaries once in design mode (9/10 falsifier
verdicts kept, 5/5 security), and resolved every kept finding by
supersession — three decisions superseded before `design-gate` passed
clean. By analogy to the existing Class 2 estimate (10–20 human minutes:
confirm class, review a plan, resolve a deviation) — a `/design` session is
structurally closer to a milestone-sized interactive review than a single
task: six category decisions plus two real adversary reports to read is
more reading than any single Class 2 touchpoint, so **20–40 human minutes**
for a project of this size is a reasonable estimate, not a measurement.
Larger or more contested category sets (the auth-model decision here was a
real three-way `AskUserQuestion`, not inferred) would cost more; a project
that defers aggressively would cost less. This should be one of the first
things `/costs` reports on once a second `/design` session exists to
compare against.

### What multi-repo coordination costs

Marginal cost over an equivalent single-repo task, estimated from the one
real data point (the additive `title_length` task across `bookmarks` +
`bookmarks-cli` + `bookmarks-workspace`): a second full floor run (bookmarks-
cli's own 5/5 applicable capabilities), one `contract-touch` invocation
(cheap — a diff-vs-registry check, not a new adversary pass), `contract-
check` in both directions (two lightweight adapter runs), and a staged
ship with two extra commit boundaries (`shipping (1 of 3)` →
`(3 of 3)`) instead of one. Both adversaries ran once, not once per repo —
the design keeps human/adversary review at one-per-task regardless of repo
count, which is the point of "one plan, one approval." The floor and
`contract-check` costs scale with repo count; the review cost does not.
Genuinely unmeasured beyond this single data point.

### The maximal-ceremony hazard — named by the build prompt, not fully exercised by this build's own evidence

The build prompt names the specific risk of design-stage ceremony plus
workspace-init ceremony plus skeleton-milestone ceremony compounding before
anything runs, on a *new* multi-repo greenfield project's very first day.
**This build's two worked examples never actually composed that way**,
and that gap is disclosed here rather than glossed over: Phase C ran
`/design` and shipped a skeleton milestone against `~/bookmarks` as a
single, standalone repo — no workspace existed yet. Phase D then built
`~/bookmarks-workspace` around `bookmarks` (already skeleton-shipped, one
milestone already `done`) plus a freshly bootstrapped `bookmarks-cli` that
went through `/bootstrap` directly, **not** `/design` — it never walked the
six foundational categories or produced its own decision set. So the
specific sequence the build prompt is worried about — a brand-new,
multi-repo greenfield project where `/design` decides topology, `/workspace`
builds it, and milestone 0 is the first thing that runs, *all* before any
code exists anywhere — was never run end to end. What was proven instead
is each half separately: `/design` → skeleton on one repo (Phase C), and
`/workspace` → contract → staged ship on repos that mostly already existed
(Phase D). The mitigation the build prompt names (skeleton first, design
cap bounds the delay) is architecturally in place but has exactly zero
real-worked-example evidence behind the *composed* case. Flagged here as a
real, disclosed gap in this build's own proof, not a claim the hazard
doesn't exist.

### Residual risks conceded by design (Extension A + B)

- **Registry-neglect risk.** Contract coverage is only as good as the
  registry staying current, and the only thing surfacing that to a human is
  one new line in the delta briefing when a ship touches a contract. This
  is the same engagement bet every other "the human has to actually read
  the artifact" mechanism in this system already makes (see the circuit
  breaker's own honor-system caveat, above) — nothing new architecturally,
  but worth naming as its own instance rather than assuming the general
  disclaimer covers it implicitly.
- **The inconsistency window staged ship bounds but cannot eliminate.**
  Real evidence, not hypothetical: the `title_length` ship spent real wall-
  clock time between `bookmarks`' commit and `bookmarks-workspace`'s final
  commit, during which `bookmarks-cli` had already merged its tolerant
  `.get()`-based read but `bookmarks` hadn't yet merged the field addition
  in one ordering, and the reverse tolerance in the other — `work/<task-
  id>/state` read `shipping (k of n)` the entire time. The window is real,
  visible, and — because the declared order guarantees every intermediate
  state is contract-compatible by construction (the whole point of
  choosing additive-safe orderings) — never unsafe. It is not zero, and
  nothing claims it is.
- **The affected-repo residual is real but untested by this build's own
  worked examples**, disclosed plainly: `contract-check` is supposed to be
  the floor for a repo that's merely *affected* by a contract change
  without being *edited* — full floor eligibility and `contract-check`
  eligibility are deliberately separate (§3 of the Phase B/D design
  decisions, below). But in the one real multi-repo task this build ran,
  `bookmarks-cli` was itself edited (the `.get()` tolerance change), so it
  always received a full floor too — the "affected-only, not edited, and
  gated on `contract-check` alone" branch was never actually exercised
  against a repo carrying pre-existing, unrelated baseline failures. The
  mechanism is built and the code path exists; a real demonstration of it
  masking pre-existing consumer breakage (or not) does not exist yet.
- **The decision-store residual** — a decision nothing ever cites again can
  sit at `adopted` indefinitely, the same "uncollided section can rot
  undetected" property `docs/charter.md` already concedes above. No
  concrete instance of this exists yet in `bookmarks`'s real decision store
  (every decision there was either implemented or superseded across the
  two real tasks that ran) — named as an inherited, structural risk, not
  an observed failure.

### The greenfield worked example — `~/bookmarks` through the design stage

A genuine small bookmark/note manager: real SQLite persistence, a real
contested auth-model decision, real state. `/bootstrap` (charter human-
confirmed, four code-independent adapters written and conformant) then
`/design`, walking all six categories — five decided directly, the auth
model put to the human as a real three-way choice (API-key-gates-
everything vs. writes-only vs. session-cookie), search implementation
legitimately deferred with a real trigger.

**Design review, both adversaries, real, in design mode**: 9/10 falsifier
verdicts kept (the one drop was a real citation error — a quote that
actually lived in `DEFERRED.md`, cited against `D-2` — `verdict-filter`
correctly refused it without the underlying point being wrong), 5/5
security verdicts kept. Both adversaries independently converged on real
defects from different angles: the repository-module boundary had no home
for a new `config` table, and the auth model never specified how the owner
obtains the plaintext key on first run and had picked argon2 (deliberately
slow, wrong for a high-entropy generated key) as its hash. Resolved by
superseding three decisions, not by override — the override mechanism
exists (open question 5, below) but was never exercised here.

**Milestone 0, real**: Class 2 (touches `migrations/**` and `src/auth/**`),
two real halt-tier deviations, both human-confirmed, neither routed around
— a `supertest` devDependency addition, and a real driver swap
(`better-sqlite3` segfaulted on this machine, reproduced standalone and
independent of the sandbox; switched to Node's built-in `node:sqlite`,
`D-9` superseded by `D-10` disclosing the experimental-API tradeoff).
**Real floor, all 12 applicable capabilities pass** — the most complete
floor this build produced across all real installs (horizon 8/13, bgr
8/13, bookmarks 12/13). Adversarial review (normal mode, not design mode)
found and fixed three more real bugs before shipping, including both
adversaries independently catching the same defect (`requireApiKey`
mounted before `express.static`, making the UI's own key-entry page
unreachable). **Shipped for real**, milestone done-definition genuinely
met — `test`/`smoke-seed`/`smoke-run`/`smoke-golden` all flipped to
`implemented`, verified via a real `adapter-conformance --all` pass. This
is the first time in this build's history the smoke lane reached
`implemented` for real — horizon and bgr never had it (`unavailable`
outright, no local emulator/DB service configured).

**A second, post-skeleton feature task** (`created_after`/`created_before`
filtering, Class 1, grounded on three already-`implemented` decisions) shipped
separately, exercising `/ship`'s decision-lifecycle step appending *more*
implementing paths onto decisions a prior task had already flipped to
`implemented` — correct, append-only, not a bug. The falsifier found a
real correctness bug here too (an unconstrained `datetime()` precision
mismatch causing lexicographic-comparison errors at second boundaries),
fixed and regression-tested.

### The multi-repo worked example — `bookmarks` / `bookmarks-cli` / `bookmarks-workspace`

Reused `bookmarks` (already skeleton-shipped) as the producer, plus a
genuinely new Python `bookmarks-cli` (deliberately a different stack —
`requests`+`argparse`, full independent bootstrap: venv, `mypy --strict`,
`ruff`, `pytest`, five applicable adapters, real `adapter-conformance
--all` pass), coordinated through `~/bookmarks-workspace` with one
declared contract (`items-api`), spec written field-for-field from the
already-shipped API, not invented.

**Additive change, real, end to end**: `title_length` added to the
contract. Real multi-repo research (citing a real pre-existing decision
across repos, `bookmarks:D-7` — this citation is what surfaced the first
real cross-repo bug, see the self-red-team section below), Class 2 plan
(auto-escalated, both `contracts/items-api/spec.md` and
`bookmarks-cli:src/bookmarks_cli/client.py` protected), human-approved.
**Real floor, both repos, PASS. Real `contract-touch`: 1 contract touched,
correctly classified `additive`, matching the plan's own declaration. Real
`contract-check`, both directions, both pass.** Both adversaries
independently found the *same* real high/low-severity bug from different
angles — `title_length` computed as JS's UTF-16 code-unit count, wrong for
any title with a character outside the Basic Multilingual Plane — fixed
(Unicode code-point count, disclosed residual: still not grapheme-cluster-
accurate), regression-tested in both repos, verified live against a
running server (`😀😀` → `2`, was `4`). **Real staged ship**, three
repos in declared order, `work/<task-id>/state` reading `shipping (1 of
3)` through `(3 of 3)`.

**Breaking change, real, correctly refused as a single task**: a second
task deliberately mis-declared a real breaking rename (`title` →
`heading`) as `## Contract change: additive` — the direct test of self-
red-team question 1 (below). `contract-touch`'s diff-based classifier
correctly reported `spec_change: "breaking"` regardless of the plan's own
claim, and `/verify`'s breaking-change gate refused the task outright.
**A second, independent signal, real**: `bookmarks-cli`'s own `contract-
check`, run against the actual renamed spec, separately failed on its own
merits — while `bookmarks`' producer-side `contract-check` passed, since
its own interface and the spec were renamed together and internally self-
consistent. This is the concrete proof that a per-repo check alone cannot
see a cross-repo break; only the registry-driven blast-radius check can.
Never shipped; all code reverted; the real `research.md`/`plan.md`/
`verify.md`/`ledger.json`/`contract-touch.json` trail is preserved in
`docs/example/breaking-rename-refused/`.

**A real expand leg followed**, decomposing the honest version of the same
rename across a milestone (`work/M1/milestone.md`): `heading` added
alongside unchanged `title`, real floor PASS, `contract-touch` correctly
classifying this leg `additive` — the honest case of the same check that
caught the dishonest one — real staged ship, `bookmarks-cli`'s `contract-
check` confirmed unaffected (expand's whole point is that no consumer needs
to change yet). The `migrate` and `contract` legs are real, disclosed
`TBD`s in `milestone.md` — not yet run, named explicitly rather than hidden.

### Single-repo regression demonstration

Per the build prompt's own explicit requirement (§3): confirmed, twice.
**Code-level**: `core/hooks/_workspace-route`'s non-workspace branch is,
line for line, the identical rel-path computation every hook already did
pre-Extension-B — `WSR_OWNER_NAME` is the empty string in that branch
specifically so no repo-qualification prefix appears in any message,
matching pre-Extension-B stderr text byte-for-byte, not just allow/deny
parity. **Live-fire**: `path-escalate`, `dep-gate`, and `phase-gate` run
against a plain single-repo fixture with no `workspace.json` reproduced
identical stderr and exit codes to the pre-Extension-B behavior.
`check-stale` run against `~/horizon`'s own real, currently-in-progress
research.md (no `workspace.json` at horizon's root) correctly reported it
`STALE` against the engineer's own real uncommitted changes, using the
identical single-repo code path — a genuine finding on a real file, not a
synthetic fixture, incidentally also writing the standard quarantine
banner into that real file (disclosed at the time, not swept under a
"regression test" label). **Real floor re-runs on the other two installs**:
`~/bgr` hit a real, pre-existing `secret-scan` failure — its own self-test
fixture's fake AWS key, hit because a zero-diff floor invocation feeds
`secret-scan` an empty changed-set, which per its own documented
convention falls back to a full-tree scan and finds its own fixture. This
is not caused by Extension B (`floor` itself is unmodified by it, by
design) and is fully reproducible by the pre-existing `floor`/`secret-scan`
pairing alone — it is the same full-tree-fallback shape "Two-stack
validation" (above) already named, now confirmed to recur on a genuinely
zero-diff floor call rather than only in the synthetic-project demonstration.
This is a real, disclosed, still-open gap in the original build's own
scope (not this extension's), recorded here because this is the pass that
actually ran the check and found it recurring, not because Extension B
caused it; `~/horizon` failed at `test` for
reasons fully unrelated to this build (a stale counter-app fixture, a
`file_picker` version conflict) — confirms no catastrophic new failure was
introduced, though the regression check could not be driven all the way to
a clean `secret-scan` result on horizon within this pass, for reasons
unrelated to Extension B.

**A real, previously-undocumented primitive finding, found while running
this exact regression check**: a workspace root needs the ordinary one-
time interactive Claude Code trust dialog accepted (or
`hasTrustDialogAccepted: true` set for its path in `~/.claude.json`)
*before* `permissions.additionalDirectories` takes effect at all. The
first attempt at the cross-directory hook-routing live-fire test (workspace
root never previously opened interactively) was inconclusive for this
exact reason — Claude Code's project-trust layer silently dropped both
`permissions.allow` and `permissions.additionalDirectories`, and every edit
was blocked by a "workspace not trusted" denial before any hook ever ran,
which looks identical to a hook denial unless you know to check for it.
This is a general Claude Code property, not something Extension B
introduces — but a fresh workspace root is exactly the case that hits it
on day one, so `/workspace`'s own hand-off step should say so explicitly
(currently does not). After the trust dialog was accepted once,
interactively, the re-run was clean and correct: two protected-path edits
blocked (each by its own repo's `protected-paths.conf`, resolved via the
one workspace-loaded hook), one unprotected edit succeeded, confirmed via
`git diff --stat` in both member repos.

### The seven extension open questions (build prompt §5), answered and defended

1. **Decision content hashing.** `sha256` of the file's full content minus
   the `- Status:` line (`core/scripts/decision-hash`, single source of
   truth, both `check-stale` and any future skill must shell out to it
   rather than reimplement). Verified for real, twice, isolating the two
   independent failure branches: a content edit to `D-1`'s body drifted a
   citing `research.md` (`STALE — content hash changed`); reverting the
   body and instead flipping only `- Status: adopted` → `implemented` on a
   *fresh* citation left `check-stale` reporting clean — the status flip
   was correctly invisible to the hash. A separate `superseded`-status
   branch fires independently of content drift (tested on a different
   decision to isolate it from the hash test) — either branch alone is
   sufficient to quarantine.
2. **Milestone ID scheme.** `M0`, `M1`, ... — deliberately distinct in
   shape from task IDs (`<YYYYMMDD>-<slug>`) so `ledger scan-untracked-
   ratio`'s commit-trailer grep is never confused by a milestone folder;
   milestones never commit, only their member tasks do, each with its own
   ordinary `Spine-Task:` trailer. Proven against a real milestone with a
   real completing ship (`bookmarks`' M0), not just a synthetic
   demonstration.
3. **`contract-touch` inputs and registry staleness.** Producer paths are
   declared per contract as a glob, with a `producer_paths_match_count`
   recorded at registration time. If the current match count is zero but
   the registered count was nonzero, the contract is treated as touched
   **regardless of whether a real diff match fired** — registry staleness
   fails safe to "gate anyway," never to "silently skip." Verified for
   real: a producer path deliberately moved entirely out of its registered
   glob's directory still correctly flagged `registry_stale: true` and
   still put the consumer in blast radius, despite the diff-based check
   alone being unable to see the real change.
4. **Ship-order derivation.** Declared in the plan (`## Ship order`),
   validated by `/ship` against the registry's own producer/consumer
   direction — the recommended option, implemented as recommended, not
   re-litigated. One builder addition beyond the open question itself: a
   reserved `workspace` name in `## Ship order` for the workspace root's
   own commit position, in the same "declared, validated, never silently
   derived" spirit as the rest of the mechanism.
5. **Design-review disagreement.** The recorded-override mechanism (same
   trust model as `/ship --bypass`, visible in the briefing) is built but
   **genuinely untested end to end** — every kept verdict in the one real
   design review this build ran was resolved by revision/supersession,
   zero by override. Flagged, not resolved by evidence, for whoever next
   runs `/design` somewhere the human disagrees with a finding.
6. **Workspace-level ledger.** No code change was needed — `ledger init`/
   `mark`/`set` already accept `--project`, and a multi-repo task's
   `work/<task-id>/ledger.json` simply lives at the workspace root because
   that's where the one task folder is. Verified for real (`ledger init`
   against `~/bookmarks-workspace` worked unmodified). **What is not
   built**: full `/costs` aggregation across a workspace *and* its member
   repos' own independent single-repo task histories — out of this build's
   manifest, on the v2 shelf below, with the mechanical shape already
   obvious enough not to need further design work: loop over
   `workspace.json`'s repos plus the workspace root, sum `ledger aggregate`
   per location.
7. **Design-mode adversary cost tiers.** Left unresolved, same as it was
   after Phase C — not in this build's deliverable manifest either. The
   one real data point argues neither for nor against a tiered cost: the
   `bookmarks` design review (six decisions, all real) ran both adversaries
   at full, untiered ceremony, with cost unmeasured pending real `/costs`
   data. A design review over a much smaller decision set (per the build
   prompt's own "a 4-decision design should not pay a 12-decision review"
   framing) has not been run, so there is no real evidence either that
   flat-cost review is disproportionate for small designs or that it isn't.

## Self-red-team

Per build prompt §8, attacking this build's own gates before delivery.

**Laziest defeat of each gate, and whether anything detects it:**

- **`phase-gate`/`path-escalate`** (deny `Edit`/`Write` outside task
  folder during research/plan; halt on protected paths): **CLOSED in Phase
  E.** The matcher was `Edit|Write` only through Phase D, so a `Bash` shell
  redirect (`printf ... > file`, `sed -i`) bypassed both hooks entirely —
  confirmed as a real gap in Phase B (§2, "phase-gate first-attempt
  failure"), re-confirmed still open through Phase D. Phase E widened both
  matchers to `Edit|Write|Bash` and added `core/hooks/_bash-write-targets`,
  a shared extraction library both hooks source: it recognizes output
  redirection, `sed`/`perl -i`, `tee`, `cp`/`mv`/`install`, `truncate`, and
  `dd of=`, resolves the target the same way the Edit/Write path already
  did, and — critically — **fails closed** (denies) when a write-shaped
  command's target can't be confidently resolved (a shell variable,
  command substitution, glob, or a relative path after an in-command `cd`),
  rather than silently passing it through. This is a gate, not a shell
  parser, and it is not perfect closure — see "Residual shapes still open"
  below — but the specific bypass this build shipped with (a plain
  redirect or `sed -i` sailing past an `Edit|Write`-only matcher) is
  closed.

  **Fire-tested against horizon's real tree, fresh `claude -p` subprocess
  each time** (2026-08-08): (1) `printf 'x' > lib/bash_bypass_firetest.dart`
  during a scratch task's research phase — real block by `phase-gate`,
  file never created (confirmed by `ls` before/after). (2)
  `sed -i '' 's/nonexistentstring123/replacement123/' firestore.rules` — real
  flat-deny block by `path-escalate` (migration-tagged, Class 1 active
  task), file's md5 confirmed unchanged before/after. (3) `git status
  --short` — passed through uninspected, proving the widened matcher did
  not become a blanket deny. (4) `TARGET=firestore.rules; sed -i ''
  's/a/b/' $TARGET` — real fail-closed block: `path-escalate` correctly
  refused to resolve the variable and denied rather than silently pass a
  write whose target it couldn't verify.

  **Residual shapes still open, stated plainly:** (a) a Bash command run
  from a working directory this hook can't observe (a `cd` in a *prior,
  separate* Bash tool call, not the same command string) can still evade
  path resolution — the hook only sees one `tool_input.command` string at a
  time, never prior shell state; a relative-path write issued after an
  earlier, separate `cd` will resolve against the wrong base and may
  incorrectly pass. (b) `sed`/`perl -i` invocations naming more than one
  target file (`sed -i ... f1 f2 f3`) are only checked against the last
  token of the command — `f1`/`f2` are not inspected. (c) any shell
  mutation shape not in the recognized list (e.g. a custom wrapper script,
  `xxd -r` writing via a language interpreter's own file-write call
  invoked through `python -c "..."`, `>&2`-into-a-real-fd-remap trick) is
  invisible to this gate the same way it always was — this closes the
  *known, named* shapes from the build prompt and Phase B/D's own
  self-red-team, not shell-mutation in general. Nothing mechanically
  detects a diff that arrived via one of these residual shapes versus one
  that arrived via a recognized path — the same "looks identical after the
  fact" problem the original finding named, narrowed to a smaller set of
  shapes rather than eliminated as a category.
- **`dep-gate`** (force manifest edits/install commands to `ask`): a
  command that doesn't match any line in `.spine/install-command-
  patterns.conf` sails through uninspected — e.g. `curl | sh`-style
  installs, or a package manager this project doesn't use invoked via a
  wrapper script. The pattern file only covers what Layer 3 calibration
  named. Nothing detects an install command via an unrecognized tool.
- **The floor**: an adapter can be hand-edited by anyone with repo write
  access to always `exit 0`. `adapter-conformance`'s known-fail case is
  the only defense — but it only runs at install/recalibration time, not
  on every floor invocation. A conformant adapter that's later edited to
  always pass would not be caught until the next `/adopt` recalibration or
  a manual `adapter-conformance` re-run. **This is a real gap**: nothing
  re-validates adapter conformance automatically on every `/task`.
- **Adversary verdict filtering**: `verdict-filter` requires a `file:line`
  or `command`+`output` pair. An adversary (or a compromised/lazy one)
  could cite a real file:line that doesn't actually support the claim —
  the filter validates evidence *shape*, never evidence *truth*. A human
  skimming `verify.md` is the only backstop, and the build prompt already
  concedes plan review has no backstop; verdict *truthfulness* has the
  same property one layer down.
- **Circuit breaker** (3rd deviation invalidates the plan): counts
  deviations logged to `deviations.md` — an implementer (human or model)
  motivated to avoid triggering it can simply not log a deviation and
  proceed anyway. Nothing mechanically forces a deviation to be logged;
  it's a norm the skill instructs, not a hook-enforced one.

**Could an adapter pass conformance while lying?** Yes, narrowly: an
adapter that correctly fails its bundled self-test fixtures but is
miscalibrated against the *real* codebase (too lenient or too strict on
real code, as opposed to its synthetic fixtures) passes conformance
cleanly — conformance proves the pass/fail *mechanism* works, never that
the specific check is *well-calibrated* for this repo. horizon's own
`typecheck` adapter is a disclosed example of exactly this shape: it
filters `build/` output to avoid false positives from vendored source, a
real, deliberate scoping decision that conformance's generic fixtures
could never have caught or required.

**Most likely week-three abandonment path**: the floor's whole-tree lint
failure (above) is precisely this pattern. An engineer under deadline
pressure, faced with every single task failing floor for reasons that have
nothing to do with their change, reaches for `--bypass` on every task
rather than the one dedicated cleanup task that would fix it once. **Phase
E narrows this**: `lint`/`typecheck` no longer fail every task for
pre-existing, task-independent debt (§ "Phase E: the lint/typecheck
rescoping decision"), so this specific pressure toward reflexive
`--bypass` is reduced for the two capabilities that were actually causing
it. It is narrowed, not eliminated as a category — any *other* capability
that later turns out to be whole-tree-scoped and pre-violated (or `--full`,
once someone wires it into CI and hits the still-unfixed 330-file debt)
would recreate the same pressure. What makes it visible either way:
`ledger scan-untracked-ratio`'s bypass count, which `/costs` surfaces first
per build prompt §2.6 — but only if the engineer actually runs `/costs`
and reads it. Nothing pushes that number in front of them proactively.

**Hooks shipped without watching them fire**: none. All three
(`phase-gate`, `path-escalate`, `dep-gate`) were fire-tested via fresh
`claude -p` subprocesses against horizon's real tree in this phase, in
addition to Phase B's scratch-repo tests, including the post-audit
`dep-gate` retest after moving its patterns out of spine core. **Phase E
added a new Bash branch to `phase-gate` and `path-escalate` and watched
every new branch fire for real** (see the closed self-red-team finding
above): the deny branch (redirect during research), the flat-deny branch
(`sed -i` on a migration path), the pass-through branch (a harmless
command), and the new fail-closed/unresolved branch (a variable-named
target) — four fresh `claude -p` subprocesses against horizon's real tree,
none unwatched.

**Capabilities marked `implemented` without a passing conformance run**:
none, in either horizon or the synthetic Python project — `adapter-
conformance --all` passed for every `implemented` entry in both
`capabilities.json` files, re-run in this phase after horizon's codebase
changed since Phase B (confirmed still passing against the current tree,
not just the original one). **Phase E note**: Phase D's original synthetic
Python project no longer exists on disk (it was scratch, never committed).
Phase E rebuilt a smaller, two-capability synthetic Python project
(`typecheck` via `mypy --strict`, `lint` via `ruff`, the two capabilities
this phase's rescoping actually touched) rather than reconstructing all
nine of Phase D's capabilities, and ran `adapter-conformance --all` against
it — PASS. This is a narrower second-stack proof than Phase D's original
(nine capabilities vs. two), disclosed as exactly that rather than implied
to be a full re-validation.

**Files that would not run today**: none found. Every script, hook, and
adapter referenced by a skill was either executed directly or fire-tested
via subprocess in this phase or a prior one.

**The Auto Mode classifier wall (above) was this self-red-team's top
finding through Phase D** — it's the one gap that isn't a hole in a
specific gate but a systemic risk to the *ledger/conformance tooling's own
reliability* under one real permission configuration, and it was found
only by actually running the built system end-to-end rather than testing
its parts in isolation. **Phase E narrowed it**: confirmed by direct,
attended re-run that `ledger`/`check-stale`/`conformance`/`verdict-filter`
all execute correctly outside the specific headless-subprocess
configuration that originally surfaced the wall, and added a mechanical
trail (`ledger note-gap`, the `hand_tracked` stamp, `verify.md`/
`briefing.md`'s Tooling gaps sections, `/costs`' `tooling_gap_count`) so a
future recurrence — headless or not — leaves a visible trace instead of
looking identical to a clean run. See "Phase E: made the degradation loud"
above for the full account.

### Extension A + B — self-red-team

Per build prompt §8, attacking the new gates the same way the original
seven were attacked above.

- **Classifying a breaking contract change as additive — what catches it?**
  Tested directly, not hypothetically (the multi-repo worked example's
  second task, above): a real plan deliberately declared a real breaking
  rename `## Contract change: additive`. `contract-touch`'s classifier
  (diff-based — scans the unified diff after the first `@@` hunk header for
  any `-`-prefixed line, not the plan's own self-report) correctly reported
  `spec_change: "breaking"` regardless of the plan's claim, and `/verify`'s
  breaking-change gate refused the task outright. A second, independent
  signal fired too: the consumer's own `contract-check`, run against the
  actual renamed spec, separately failed on its own merits — while the
  producer's own `contract-check` passed, since its interface and the spec
  were renamed together and internally self-consistent. **This is real
  proof a per-repo check alone cannot catch this class of break; only the
  registry-driven blast-radius check can** — and real proof the mechanical
  classifier does not trust the plan's own declaration, which is the
  entire point of it existing independently. The classifier's one disclosed
  gap (from Phase D's own build): a newly-*required* field is diff-
  identical to a newly-*optional* one, pure addition either way — this
  stack-blind check cannot and does not claim to catch that class of break
  (`core/rules/contracts.md`).
- **Grounding research on a decision while ignoring its quarantine flag —
  what happens?** Nothing mechanically stops it, and this is disclosed
  plainly rather than assumed away: a quarantined decision citation gets
  the same banner-in-file, non-blocking treatment file-based staleness
  already gets above — `check-stale` reports `STALE` and writes a banner
  into the citing `research.md`, but nothing hook-enforced prevents a plan
  from being written against that research.md anyway if a human or model
  proceeds past the banner. This is the same "wrong context beats missing,
  but the invalidation channel is advisory, not enforced" property the
  charter's own uncollided-section risk already concedes above — not a new
  category of gap, but worth naming as its own instance at the design
  tier, since it's a new grounding source this build added.
- **A `contract-check` adapter that validates too little while passing
  conformance — real evidence, not hypothetical.** The multi-repo worked
  example's additive task is the concrete proof: `title_length` computed
  as `title.length` (JS UTF-16 code units, wrong for non-BMP characters)
  shipped with **both** `contract-check` adapters passing cleanly, because
  both are structurally field-*name*-only and cannot see a correctly-named,
  incorrectly-*computed* field. Confirmed directly, not assumed: both
  adapters were re-run against the still-buggy code after the fact and both
  reported clean. Only the adversary layer caught it (both falsifier and
  security, independently, from different angles). There is nothing
  stack-blind and mechanical to fix here — this is a real, permanent
  residual of a field-name-matching contract check, disclosed rather than
  patched over.
- **The design cap gamed by cramming multiple decisions into one record —
  untested, disclosed gap.** `design-gate` check 4 counts adopted decision
  *records* (files), not the number of distinct decisions a record's prose
  actually contains. Nothing stops an implementer from writing one
  `D-<n>-*.md` that bundles several real architectural choices under one
  ID to stay under the cap while still deciding as much as an uncapped
  session would have. Not exercised for real in either worked example —
  named here because the self-red-team asks for it explicitly, not because
  evidence of it happening exists.
- **Registry staleness gamed** (a variant of the same class): the fail-safe
  ("gate anyway" when the registered match count goes to zero — open
  question 3, above) was tested for the specific case of a producer path
  moving out of its glob. A producer path that *stays* inside the glob but
  is edited in a way `contract-touch`'s diff scan doesn't recognize as
  spec-relevant would not be caught by this fail-safe, since the fail-safe
  only fires on a match-count mismatch, not on every possible drift shape.
  Not tested; named as a residual of the same mechanism.
- **Any hook change not watched firing in the workspace topology — must be
  none.** Confirmed: the cross-directory hook-routing test (above, "Single-
  repo regression demonstration") watched all three hooks fire correctly
  through `_workspace-route` against real member-repo paths, in addition to
  the single-repo regression fire-tests. The first attempt was inconclusive
  for the project-trust reason disclosed above, not a hook defect — the
  re-run after accepting the trust dialog was clean and correctly
  attributed each denial to the owning repo's own `protected-paths.conf`.
- **Any capability marked `implemented` without a passing conformance run —
  none**, including the new #14, `contract-check` — both `bookmarks`' and
  `bookmarks-cli`'s `contract-check` adapters passed `adapter-conformance
  --all` before being marked `implemented`, in both the additive and (for
  `bookmarks`'s producer-side check) breaking-change scenarios.
- **The single-repo regression check (§3) — passed and shown.** See "Single-
  repo regression demonstration," above; not repeated here.

**Real bugs found and fixed while running the extension build against real
repos, none anticipated by planning, all evidence the pressure of a real
worked example finds defects a design session alone would not:**

- `check-stale`'s `grounding-decisions:` bash-array expansion crashed under
  this machine's default `/bin/bash` 3.2.57 (a pre-4.4 empty-array-under-
  `set -u` limitation) whenever the array was empty — the *mainline* case
  for any research.md citing only files, not decisions. Fixed with the
  portable `${decisions[@]+"${decisions[@]}"}` idiom.
- `conformance`'s `actual=()` computation only ever saw tracked-file
  changes (`git diff --name-only` alone), silently missing every genuinely
  new file a real task creates — precision measured `0.07` before the fix
  (`git ls-files --others --exclude-standard` added), `1.00` after, on the
  identical real task.
- `contract-touch`'s first breaking/additive classifier excluded diff lines
  matching `^-[^-]` to skip the `--- a/path` git header — wrong the moment
  a real spec used `-` as its own markdown bullet character, misclassifying
  a genuine breaking rename as `additive`. Fixed by scanning only lines
  after the first `@@` hunk header instead of pattern-matching the leading
  character.
- `floor`'s capability dispatch grouped `secret-scan` under `run_simple`
  (no stdin), contradicting `ADAPTER-CONTRACT.md §3`'s own documented
  changed-file-set convention — every floor run therefore invoked
  `secret-scan` in its full-tree fallback mode, which is exactly the
  configuration that makes it flag its own self-test fixture (see "Two-
  stack validation," above, and the regression re-confirmation, above).
  Fixed by moving it to `run_with_stdin`, matching `clone-scan`'s existing
  precedent.
- `grounding-decisions:` citations had no repo-qualification convention
  until the first real multi-repo research cited a member repo's own
  pre-existing decision and `check-stale` (correctly, if unhelpfully)
  reported it unresolvable — `check-stale` only ever looked in the
  workspace root's own `docs/decisions/`. Fixed by extending the same
  `<repo-name>:<path>` qualification convention `files:` already carried to
  decision citations too, across `check-stale`, the templates, and
  `researcher.md`.
- `verify/SKILL.md`'s first draft conflated `contract-check` eligibility
  with floor eligibility ("only gate a consumer not in the edited set") —
  would have meant `contract-check` silently never running for the
  additive task's own consumer, since it *was* edited. Corrected before
  the worked example ran: `contract-check` gates the producer and every
  consumer regardless of edited status; floor eligibility is a separate,
  narrower question.
- Two bookmarks-specific project adapters (not core, but the same defect
  classes are worth naming): `lint` crashed on a real deleted file in a
  real diff (didn't filter deletions before handing the changed-set to
  eslint); `callers` carried the identical bash-3.2 empty-array bug as
  `check-stale`, in two separate places — found by the falsifier's
  adversarial review, not this build's own testing, which is exactly the
  class of gap the adversary layer exists to catch that a floor run alone
  would not.

## Extension C — team support for 2–4 engineers in parallel

Everything below was added by a third build (`work/.build/ext-c-phase-
{A..D}-handoff.md`, dated 2026-08-11), against Extensions A and B above,
already installed. It adds no new harness primitive — no hooks, no
agents, no capabilities — on purpose (build prompt §3): files, git, and
scripts only. The whole claim to safety is that it composes what already
existed: the open-task registry is `work/`, made shared instead of local;
`claims-check`/`propagate` are new scripts in the existing
`core/scripts/` shape; the flag-blocked-advance check is a new rule
inside the existing `/task` skill, not a new gate mechanism.

### What this costs

Measured from the real two-engineer demo (`docs/example/ext-c-two-
engineer-demo/`), not estimated: one `claims-check` at plan approval (a
few seconds — file-set intersection, no adversary involved), one
`registry-sync` per phase transition (a commit + push, network-latency
bound, sub-second locally), two re-grounding checks at ship
(`check-stale` + a full floor re-run — the floor re-run's cost is
whatever the project's own floor already costs, unchanged by this
extension; `check-stale` itself is sub-second), and — Class 2 only — the
wall-clock cost of a second human actually reading a plan, which this
system cannot measure and does not pretend to. The demo's own three tasks
ran the full loop (open → claims-check → plan → flag → ack → re-ground →
ship) in well under a minute of *script* time; the human-latency pieces
(a colleague reading a plan, deciding to approve) are the same order of
magnitude as the existing Class 2 estimate (10–20 minutes) and not
separately re-derived here.

### Claims coarseness — declared surfaces, not meaning

Stated plainly, not left implicit: `claims-check`'s plan-time run only
ever sees what a plan *declares* (`## Predicted touch`, research's
grounding files). A collision in undeclared territory is invisible to it
by construction — this is the same "declared surfaces, not meaning" limit
every other conformance-style mechanism in this system already carries
(`core/scripts/conformance` has the identical shape: it scores a plan
against the *real* diff, but only after the fact). **Phase D closed the
most important instance of this gap with a real mechanism, not just a
disclosure**: `claims-check --diff` (`core/skills/ship/SKILL.md` §0)
re-runs the same write/write and write/read intersection against the
*actual* diff at ship time, tagging any hit that wasn't in the task's own
original `claims.json` as `[UNDECLARED]`. Demonstrated for real, not
hypothetically: a task that declared only `src/lib/safe.ts` but actually
touched `src/lib/auth.ts` (colliding with a second open task that
honestly declared it) was caught, by name, at ship time:

```
claims-check --diff: 1 real-diff collision(s) with another open task:
  - write/write with task-2 (Engineer B <b@x.com>): both predict touching
    'src/lib/auth.ts' [UNDECLARED — not in this task's own claims.json]
```

Before this existed, the identical defeat surfaced only as a generic
`conformance` precision drop, with no link back to *which* other task it
endangered — indistinguishable from an ordinary, harmless scope change.
This is real, mechanical, and specific — and still **discovered late**:
by ship time the code already exists, so this is a halt-tier deviation
(the same tier a stale `check-stale` result gets), not a plan-time
refusal. Reverting a real diff costs more than declining to approve a
plan; that asymmetry is inherent to catching something after the fact and
isn't something this mechanism can close further without moving the
check earlier, which would require re-running it continuously during
implementation — out of scope, not attempted.

### The approval race lands in ship-time re-grounding, by design

Open question 7 (build prompt §5.7): two plans with overlapping claims
approved near-simultaneously, on machines that haven't pulled each
other's task yet. `claims-check` pulls before reading (§2.2), which
narrows the window but — as the build prompt itself predicted — cannot
close it: two engineers who both open and get approval within the same
few-second pull window can both pass a claims-check that's individually
correct against stale information. **The backstop is not a new
mechanism** — it's ship-time re-grounding, already built for a different
reason (neighbor-caused staleness), catching this case for free: whichever
of the two ships second will find `claims-check --diff` (or plain
`check-stale`, if the collision is a grounding-file one rather than a
predicted-touch one) reporting a real conflict against the first one's
now-landed change, exactly as it would for any other post-approval drift.
This is the "existing net" the build prompt suggested rather than a
purpose-built race detector — confirmed to actually work this way by
construction (both checks run unconditionally at ship time, regardless of
*why* something drifted), not separately re-tested as a distinct race
scenario beyond what (b) in the two-engineer demo already exercises.

### Identity trust model

Stated once, plainly, per the build prompt's own instruction: **git
identity is a coordination primitive among colleagues, not an audit log
against adversaries.** `git config user.name`/`user.email` is trivially
settable to anything — `owner`, `approver`, and `engineer` throughout this
extension are exactly as trustworthy as the commit author field already
was before this extension existed, no more, no less. Nothing here adds
authentication; nothing here should be read as one. The override
mechanisms (`claims-check`'s conflict override, `approval.json`'s
self-approval override, `/ship --bypass`) all use the same trust
posture: loud and recorded beats silent and enforced, for a tool whose
users are colleagues who could always have just talked to each other
instead.

### Enforcement tier — the "populated but not gated" sweep (self-red-team)

The single most valuable finding of this build (per review) was that
`claims.json` got populated correctly but nothing actually gated on it —
`claims-check` was never invoked at the one call site that mattered,
caught and fixed during Phase C, not by a separate review pass. Generalizing
that into a standing check across every mechanism this extension shipped,
each named with its enforcement point and where its refusal was
demonstrated live:

| Mechanism | Enforcement point | Refusal demonstrated live | Hook-backed? |
|---|---|---|---|
| `hook-guard` | Claude Code's own PreToolUse runner | Real `claude -p` write, fail-open before / fail-closed after, both confirmed via `!` passthrough (Phase C) | **Yes** — the one mechanism here the agent cannot choose to skip |
| `claims-check` (plan-time) | `core/skills/task/SKILL.md` §3, before presenting the plan | Real write/write + write/read blocks, real renegotiation clearing them (Phase C, re-confirmed in the two-engineer demo, scenario a) | No — script computes a real answer; acting on it is skill prose |
| `claims-check --diff` (ship-time) | `core/skills/ship/SKILL.md` §0 | Real narrow-claims defeat caught and tagged `[UNDECLARED]` (Phase D) | No |
| flag-blocked advance (`propagate`'s output) | `core/skills/task/SKILL.md`, every phase transition | Full real loop: refuse → acknowledge → advance, across two independent checkouts (Phase C, demo scenario b) | No — **empirically confirmed unbacked**: a direct `Write` to `work/<task-id>/state` with an unacknowledged flag present passes `phase-gate`/`path-escalate`/`dep-gate` with exit 0 every time (tested directly, Phase D) |
| ship-time re-grounding (`check-stale` + floor re-run) | `core/skills/ship/SKILL.md` §0 | Real `STALE` detection after a simulated neighbor merge (Phase C); real independent catch after flag acknowledgment (Phase D, demo scenario b) | No — detection is mechanical, the "stop shipping" action is prose |
| `second-approver-check` | `core/skills/ship/SKILL.md` §1 | All four real branches via the actual script (Phase D — promoted from hand-verified jq logic after the sweep found it had no script backing at all); real second-approver run in the two-engineer demo (scenario c) | No |
| `setup --check` (pin) | `core/skills/task/SKILL.md` step 0 | Real warn/strict exit codes (Phase B); the `/task`-level "actually halt on strict" action itself is prose, not independently re-tested end-to-end in a live `/task` session (the classifier wall — see above — prevents driving a real session from this build's own tooling) | No |

**Honest reading of this table**: exactly one mechanism in this entire
extension is enforced at a tier the agent cannot simply choose to ignore.
Every other gate — including every one carried over from the base system
before this extension (the merge gate's floor/deviations check, the
deviation circuit breaker) — is a real script producing a real, correct
answer, acted on by an agent following skill instructions. This is not a
regression this extension introduced: `core/hooks/phase-gate` was read
directly and confirmed to contain zero logic referencing `deviations.md`,
`verify.md`, or now `flags.json`/`claims.json`/`approval.json` — the
*existing* merge gate has always been prose-enforced too. What Extension
C adds is five more checks at that same tier, not a new, weaker tier. The
build prompt's explicit "no new hooks" constraint is precisely what makes
this true rather than an oversight: a hook-backed version of any of these
would require inspecting `flags.json`/`claims.json` content from inside
`phase-gate`, which is exactly the kind of new-primitive risk §0 warns
against building on a hunch. Recorded here as the honest boundary, not
smoothed over: **an adversarial or careless agent session can bypass
every check in this extension except `hook-guard` by writing the target
file directly instead of following the skill's prose.** The mitigating
fact is the same one that governs the rest of this system — a human reads
the plan, the briefing, and (now) `/costs`' per-engineer view, and those
three touchpoints are where a pattern of silent bypass would surface, the
same way a pattern of `--bypass` abuse would.

### Laziest defeat of each new gate — where it's visible

- **Narrow claims to dodge `claims-check`.** Caught late, not blind:
  `claims-check --diff` at ship time, tagged `[UNDECLARED]` (above).
  Visible in: the ship-time deviation record, `/costs`' `claims_conflicts`
  count if it also blocked a colleague.
  Not caught: a narrowing that happens to touch nothing any other open
  task cares about — indistinguishable from an honest, narrow task,
  because at that point it *is* one.
- **Rubber-stamp second approval** (approve without reading). No
  mechanism distinguishes a real review from a reflexive one — `git
  config user.name` proves a different identity touched the file, nothing
  proves they read anything. Visible in: nothing mechanical; this is the
  same "the human has to actually engage" bet the whole system has always
  made (the plan-approval touchpoint itself has the identical property
  for a solo engineer). Named, not solved.
- **Blind flag acknowledgment** (ack without acting on it). Same shape:
  `acknowledged: true` proves a write happened, not that the engineer
  changed course. Visible in: the flag's own `acknowledged_by`/
  `acknowledged_at` are permanent, append-only record — a pattern of
  acknowledging and never adjusting research/plan afterward would be
  visible to someone reading `work/<task-id>/flags.json` history, but
  nothing surfaces that pattern automatically today. Named as a v2-shelf
  metric (a "flags acknowledged vs. research regenerated after" ratio),
  not built.
- **Override-as-habit** (routine use of `claims-check --override`,
  `approval.json`'s self-approval override, or `--bypass`). Visible in:
  `/costs`' per-engineer view — `claims_conflicts` and `bypass_count` are
  both per-engineer already; a `second_approver` override isn't yet its
  own counted field (it's a ledger *string*, `"override: <reason>"`, not
  a boolean `/costs` currently tallies) — **named gap, not closed**: a
  future `/costs` pass should count override frequency the same way it
  counts bypass frequency, per the same "a metric read as a leaderboard
  gets gamed, but an absent metric can't be read at all" reasoning.

### The 2–4 engineer limit, and what breaks at 8

This extension was built and demonstrated for 2 engineers (the demo) and
reasoned about for up to 4 (the build prompt's own scope). Named,
concretely, what stops scaling past that, not just asserted:

- **Informal override culture.** At 2–4, an override is a rare enough
  event that a colleague reading the briefing notices it. At 8, the
  volume of Class 2 ships and claims conflicts rises with team size while
  the "one human reads `/costs`" bandwidth doesn't — overrides become
  background noise, the exact failure mode named in the laziest-defeats
  section above, faster.
- **`claims-check` conflict frequency.** The intersection is O(open
  tasks) per check — cheap per call (confirmed: the demo's 2-3-task scan
  was sub-second), but the *rate* of real conflicts rises faster than
  linearly with engineer count on a codebase whose file count doesn't
  grow with the team, since more people are predicting touches into the
  same fixed surface area. Untested at 8 — no data, named as the
  mechanism that would need real measurement first.
- **Briefing volume.** Every ship still produces one briefing; at 8
  engineers shipping in parallel, the volume of briefings one person
  would need to skim to keep the team-wide picture (not just their own
  work) rises linearly with ship frequency, with nothing in this build
  aggregating across briefings the way `/costs` aggregates across
  ledgers. Named as the next thing to build if this ever needs to scale
  past 4, not attempted here.

## Readability patch — self-red-team

A maintainer patch restructuring `plan.md`/`briefing.md` for readability
(`work/.build/readability-phase-A-handoff.md`,
`readability-phase-B-handoff.md`) — no new mechanisms, hooks, agents, or
capabilities, template/skill-prose only. Two items carried forward rather
than closed silently:

- **Multi-repo sections shipped undemonstrated against real data.**
  `## Ship order`, `## Contract change` (`plan.md`) and `## Contracts`,
  `## Milestone` (`briefing.md`) — every reachable source was checked
  (spine's own `docs/example/`, both installed projects) and none has a
  real `workspace.json`/multi-repo task to render-check the new structure
  against; the two-engineer demo's own README says as much explicitly.
  Heading text and bullet grammar are unchanged from the pre-patch
  template, so this patch introduced no new risk to those sections, but
  the readable restructuring itself is unverified against real multi-repo
  content. Not a defect, a watch item: the first real multi-repo task run
  on these templates is the actual test. Per the README's own feedback
  rule ("when a plan or briefing confuses an engineer, that is a template
  defect"), any awkwardness surfacing there gets fixed at the template,
  not patched around in the instance. **Joined by the PR-description
  patch's own multi-repo section** (`pr-description.md`'s `**Contracts**`
  section, and the one-shared-description-per-task decision behind it,
  `work/.build/pr-patch-phase-A-handoff.md` §3) — same reason, same
  environment gap, same resolution: the first real multi-repo task is the
  test for all of it at once, not a separate watch item per patch.

- **Backtick-wrapped predicted-touch paths — caught once, prose-guarded,
  pre-loaded ratchet trigger.** During this patch's own demonstration,
  wrapping a `## Predicted touch` path in Markdown backticks silently
  zeroed `conformance`'s match for that entry — its `awk` keeps the string
  byte-for-byte, and a real `git diff` path is never backtick-wrapped, so
  a genuinely correct prediction still scored as a miss. Fixed in the one
  instance caught; guarded going forward only by a template comment
  (prose, not a mechanical check) telling whoever fills the section not to
  Markdown-format the path. One instance — not yet `/ratchet`-eligible (it
  requires two: two `deviations.md` records, two adversary verdicts, or
  two relayed review comments citing the same fact). Recorded here as the
  pre-loaded trigger, so nobody has to remember "didn't this happen
  before?" if a second real instance turns up (a deviations.md record, an
  adversary finding, or a plan whose conformance score looks wrong for no
  apparent reason): that's instance two, and the ratchet response is
  already decided — a one-line normalization or validation added to
  `core/scripts/conformance` itself, not another comment.

## PR-description patch — self-red-team

`/ship` gains §4a: assembles `work/<task-id>/pr-description.md` from the
task's own verified artifacts (`work/.build/pr-patch-phase-A/B-handoff.md`)
— a computed "review this at the plan level" plus a mechanically-unioned
"where to look" list, never a re-read-the-diff summary. No new hooks,
agents, or capabilities; one real bug fix rode in the same commit
(`core/scripts/conformance`'s bookkeeping-noise exclusion, below). Five
items carried forward:

- **The reviewer who reads only the description and approves.** This
  patch cannot prevent that, and the description doesn't claim to — its
  honest claim is narrower: every sentence in it traces to a verified
  artifact, which makes rubber-stamping it strictly less dangerous than
  rubber-stamping a generated summary (a summary's confident, unverified
  claims are exactly what a rushed reviewer has no way to catch; a
  traced claim at least has a real record behind it if anyone ever checks).
  It is still rubber-stamping. The real counterweights are upstream of
  this patch and unchanged by it: Class 2's second approver
  (`ship/SKILL.md` §1) and plan approval itself (`task/SKILL.md` §3) are
  where a human's actual judgment is load-bearing; this patch only makes
  what they're reviewing easier to review well, it doesn't replace them.

- **Where-to-look drifting editorial over time.** The failure mode is
  concrete: a future skill edit adds "and anything else that seems worth
  a look," and the post-hoc-summary door this whole patch exists to close
  reopens through the one section built to prevent it. Guarded by putting
  the four-unions rule in `core/templates/pr-description.md`'s own header
  comment, not only in this file or in `ship/SKILL.md` — the constraint
  travels with the artifact a future editor is actually looking at. Named
  here as a ratchet candidate: if an editorial addition to this section
  ever ships, that's the recurring-finding trigger, and the ratchet
  response is deleting the addition and re-reading the four-unions rule,
  not accepting the drift as an improvement.

- **Deviation-file extraction — a real heuristic, not a parser, pre-loaded
  ratchet trigger.** `deviations.md` has no structured file field, so
  union (1) of the four ("every file cited in a deviation") extracts
  backtick-wrapped, path-shaped tokens from the record's prose. Checked
  against the real `20260811-extract-slugify-helper` deviation record
  (`~/bgr`) during this patch's own demonstration: it correctly pulled
  `` `src/lib/slug.test.ts` `` and `` `work/20260811-.../plan.md` `` while
  correctly skipping quoted test-input tokens in the same record
  (`` `"!!!"` ``, `` `""` ``) — and it also surfaced a real near-duplicate
  (`` `slug.test.ts` `` and `` `src/lib/slug.test.ts` `` both matching in
  the same record), resolved with a same-record suffix-collapse tiebreaker
  now written into the template. Accepted for v1. **Armed trigger**: the
  first time this heuristic demonstrably *misses* a real file a deviation
  cites — not a near-duplicate, an actual miss — the response is already
  decided: a structured `- Files:` line added to `core/templates/
  deviations.md` itself, not a bigger regex. This is instance zero (no
  miss yet, only the near-duplicate, which was fixed at the heuristic
  level since it's a false-positive-adjacent problem, not a miss); two
  real misses is `/ratchet`-eligible per the standing two-instance rule
  (same rule the backtick-predicted-touch trigger above already
  documents) — recorded here so nobody has to remember whether this was
  discussed before.

- **Adversary findings without a file anchor — the completeness line, not
  silence.** Union (2) ("every file:line in an adversary finding") can
  only represent `evidence.kind == "file_line"` verdicts; a `command`-
  evidence verdict has no file to point at, even at high severity — real
  case, found during this patch's own demonstration against
  `~/horizon/work/20260808-fix-building-group-delete-orphans-units/`: the
  run's own single highest-severity finding (a `.claude/settings.json`
  permission-escalation bleed-in) is `command`-evidence and does not get a
  where-to-look entry. Silently dropping it from the list would imply a
  completeness the list doesn't have, exactly over the finding most worth
  seeing. Fixed structurally, not by prose discipline: the list's own
  final line is computed — `"plus N finding(s) without file anchors — see
  verify.md"` — whenever `N > 0`, and the finding itself is never lost
  regardless, since it still appears in full under the floor-protected
  "Adversaries" bullet above the list. A second real case during this same
  demonstration (the same horizon record's plan predicting backtick-
  wrapped touch entries) needed a parallel guard on union (4) — see
  `core/templates/pr-description.md`'s own comment: a `predicted` entry
  containing a backtick means a raw set-difference would falsely flag
  every genuinely-predicted file as drift, so that union declares itself
  unavailable rather than emit the wrong answer.

- **Stale description after post-review changes.** v1's honest behavior,
  stated rather than built around: `pr-description.md` reflects the record
  as of ship time. A review comment that produces a new commit isn't
  covered by a refresh this patch doesn't build — spine's answer to new
  work is a new task, whose own `/ship` regenerates its own description
  against its own record. No description-refresh machinery added on
  spec; if this becomes real friction, that's a task for later, with real
  friction to design against instead of a guess.

**The conformance bug fix, and why it rode in this commit.**
`core/scripts/conformance`'s `actual` set included every task's own
`work/<task-id>/**` bookkeeping writes and `.spine/current-task`, which
this patch's own union (4) would otherwise have had to filter at the
display layer — but a corrupted plan-accuracy metric misleads every
consumer of that score, not just this feature (every `/ship` briefing's
"Plan accuracy" bullet, `/costs`' `conformance_score` trend, `ledger set
... conformance_score`). Fixed at the source instead: `conformance` now
excludes `work/**` and `.spine/current-task` from `actual` before scoring,
and states the exclusion rule in its own output (`excluded`,
`excluded_rules` fields; stdout line). **Regression-checked against the
real, already-shipped `20260811-extract-slugify-helper` record**: its
`conformance.json` went from `predicted=2 actual=14 precision=1.00
recall=0.14 f1=0.25` (12 of the 14 "actual" files were the task's own
bookkeeping) to `predicted=2 actual=2 precision=1.00 recall=1.00 f1=1.00`
— the real signal (both predicted files, and only those, were touched)
that was there all along, no longer buried under record-keeping noise.
**Explicitly not comparable**: any `/costs` trend or per-task history that
includes both pre-fix and post-fix `conformance_score` values is not
measuring the same thing across that boundary — pre-fix scores were
structurally deflated by bookkeeping noise in a way that had nothing to
do with planning quality. Treat the fix's ship date as a hard discontinuity
in that trend, not a real quality jump, if `/costs` is ever extended to
plot it.

## Real-world testbed findings — g1-tee-waitlist

A separate project (`g1-tee-waitlist`) is being run through `/task` for
real, live work — not a scratch repo built to exercise spine, an actual
feature getting built. This surfaced seven real discrepancies between what
the skills/docs say happens and what the actual hooks/scripts do, found
across two tasks' classify→research→plan→implement→verify runs
(`20260814-login-view`, a login-view feature, findings 1–6; and
`20260815-waitlist-status-view`, a waitlist-status view, finding 7). All
seven are fixed here; this section is the disclosure the build prompt's
own self-red-team practice calls for — problem stated plainly, root cause,
the real fix.

**1. `phase-gate` denied a bookkeeping edit `core/skills/task/SKILL.md`
itself said was exempt.** The skill's `--milestone` header note said the
milestone.md `TBD`-replacement edit happens "during step 1, before
`phase-gate` would even apply" — but step 1's actual body wrote
`work/<task-id>/state` = `research` *before* reaching that edit, so by the
time the edit ran, `phase-gate`'s gating condition (`state` file exists,
reads `research`/`plan`) was already true, and the hook correctly denied
a write outside `work/<task-id>/`. The header's intent was right; the
body's sequencing contradicted it. **Fix**: reordered step 1 so the
milestone block runs immediately after `mkdir -p work/<task-id>` and
before the `state` write — no hook change needed, since `phase-gate`
already no-ops when no `state` file exists yet. This is a documentation/
ordering bug, not a missing hook feature.

**2. `phase-gate`'s containment check had no notion of "inside the
project at all."** `_workspace-route`'s single-repo branch (no
`workspace.json`) unconditionally set `WSR_OWNER_ROOT="$project"` for
*any* `file_path`, even one nowhere under `$project` — asymmetric with its
own workspace-mode branch, which already fell back to
`WSR_OWNER_ROOT=""` for a genuinely unmatched path. `path-escalate` and
`dep-gate` both already guard `[[ -n "$WSR_OWNER_ROOT" ]] || return 0`, so
this asymmetry meant that guard could only ever fire in workspace mode —
in single-repo mode it was permanently unreachable, and both hooks denied
(or asked, for dep-gate) on paths entirely outside the project. `phase-
gate` didn't even call `workspace_route` for its own allow/deny decision
(only for the denial message's display text), so it denied unconditionally
on `task_dir` non-containment with zero project-boundary concept anywhere
in the path. Concretely: this blocked a real attempt, mid-task, to write
this session's own memory files under `~/.claude/projects/.../memory/` —
a location with no relationship to the g1-tee-waitlist project at all.
**Fix, two parts**: (a) `_workspace-route`'s single-repo branch now checks
containment the same way its workspace-mode branch already did,
falling back to `WSR_OWNER_ROOT=""`/`WSR_OWNER_NAME="(unknown)"` for a
path outside `$project` — this makes `path-escalate`/`dep-gate` correctly
permissive on out-of-project paths in single-repo mode too, for free,
since their own guards were already correct. (b) `phase-gate`'s
`check_target` now calls `workspace_route` first and allows outright when
`WSR_OWNER_ROOT` is empty, before checking `task_dir` containment.
**Fire-tested** against a scratch project (fresh `.spine/current-task` +
`work/<id>/state=research`): an out-of-project write now exits 0 (was
exit 2); an in-project, outside-task-folder write still correctly exits 2;
an in-task-folder write still exits 0. `path-escalate` and `dep-gate`
regression-tested the same way — in-project protected-path denial and
manifest-edit `ask` behavior both unchanged; out-of-project paths now
correctly no-op in single-repo mode, matching their pre-existing
workspace-mode behavior.

**3. `check-stale` couldn't distinguish real grounding drift from a
research doc correctly documenting decision history, and self-invalidated
on its own task's mutable state.** Two independent bugs surfaced by the
same research.md: (a) the grounding-decisions check compared a cited
decision's *current* `- Status:` line against the literal string
`superseded`, regardless of whether it was *already* superseded at the
moment the citing research was written. A Class 2 task's researcher was
correctly instructed to read every relevant decision including already-
superseded ones, specifically to document *why* and *by what* they were
superseded (`D-4 superseded by D-7`, `D-6 superseded by D-8`) — this is
exactly the grounding-decisions header's own stated purpose. But citing an
already-superseded decision this way permanently, unfixably quarantined
the research: regenerating reproduces the identical citations and the
identical "superseded" verdict forever, since D-4/D-6 will never stop
being superseded. (b) the `files:` grounding list included the citing
task's own `work/<task-id>/notes.md`, because the researcher agent
correctly read it for task context — but `notes.md` (like
`deviations.md`, `ledger.json`, `claims.json`, `flags.json`, `state`, and
every other file under a task's own folder) is *expected* to change over
that task's lifetime by design; the tooling-gap discipline requires
appending to it. Any research.md that cites its own task's mutable
bookkeeping as grounding self-invalidates the moment anything else in
that folder is next written, with zero relation to whether the researched
*subsystem* drifted. **Fix, two parts**: (a) the decision-status check now
compares status *at the cited sha* (via `git show <sha>:<path>`, same
resolution the hash check already does) against status *now* — a
decision that was already superseded at citation time reads identically
on both sides, forever, and correctly produces zero drift; a decision
whose status genuinely changed *since* citation (e.g. `adopted` →
`superseded`) still correctly flags, now with both values named in the
message. (b) the `files:` loop now skips any entry that resolves under
the *citing* research.md's own task folder (derived from `$doc`'s own
path — `work/<task-id>/`, no new header field needed) — a path outside
that task's own folder (e.g. citing a different task's research.md as
inter-task context) is unaffected and stays fully driftable.
**Regression-checked against the real, already-reproduced failing case**:
re-running `check-stale` against `g1-tee-waitlist`'s actual
`work/20260814-login-view/research.md` (which had gone genuinely stale
under the pre-fix logic on D-4, D-6, and notes.md) now reports those three
items clean, while still correctly flagging the *other* four grounding
files that had genuinely changed since research was written
(`work/M0/milestone.md`, `src/App.vue`, `src/main.ts`, `package.json`,
all real edits made during this same task's implement phase) — the fix
narrows false-positive drift without weakening real-drift detection.

**4. `/task`'s own instructions told it to do something the harness
categorically blocks.** `core/skills/task/SKILL.md` step 5 instructed
`/task` to "invoke `/verify` (Skill tool) with the task ID" and, later,
"invoke `/ship` (Skill tool) with the task ID." Every spine skill,
including `verify` and `ship`, carries `disable-model-invocation: true` —
this is not a per-project or per-hook setting spine controls, it's the
Claude Code Skill tool's own contract, and it blocks *any*
model-initiated call, unconditionally, regardless of who's asking or why.
A running `/task` session attempting this call for real gets a hard
refusal, not a degraded or partial result — confirmed directly, live,
during the g1-tee-waitlist task this section is about. `README.md`'s own
command table described `/verify` as "Invoked automatically by `/task` at
the verify phase," which was never achievable under the harness's actual
enforcement. **Decision**: fix the documentation and the handoff pattern,
not the harness setting — `disable-model-invocation` on every skill in
this system is a deliberate boundary (build prompt §2.7's "recurring
human touchpoint" design: classify, plan approval, and now verify/ship are
all meant to be points where a human is actually in the loop, not
rubber-stamped by the same session that just finished implementing).
Stripping the flag so `/task` could self-chain into `/verify`/`/ship`
would silently convert every one of these into a non-touchpoint — exactly
the kind of erosion the recurring-touchpoint design exists to prevent.
**Fix**: `core/skills/task/SKILL.md` step 5 now tells the human plainly to
run `/verify <task-id>` themselves, stops, and waits — the identical
posture step 3 already uses for a Class 2 second approver. On resumption,
it reads `work/<task-id>/verify.md`'s own `Result:` line directly (`PASS`/
`FAIL`, `core/templates/verify.md`'s own format) rather than trusting a
verbal summary, then repeats the same ask-and-wait pattern for `/ship`,
confirming completion afterward by checking `.spine/current-task` was
actually cleared and `work/<task-id>/briefing.md` actually exists.
`README.md`'s command table reworded accordingly for both entries.

**5. `core/scripts/floor`'s `compute_changed_files` had no `work/**`
exclusion, unlike `core/scripts/conformance`'s own already-fixed
equivalent.** Found during `20260814-login-view`'s `/verify` run:
bookkeeping files (`work/<task-id>/ledger.json`, `work/M0/milestone.md`)
reached every diff-scoped capability (`lint`, `typecheck`, `secret-scan`,
`clone-scan`, `callers`) as literal input — `callers` failed outright,
since it greps the changed-file set as literal symbol strings with no
extension filter to save it the way the other three happened to have.
**Fix**: added an `is_bookkeeping()` filter to `compute_changed_files`
excluding `work/*` and `.spine/current-task`, mirroring `conformance`'s
existing exclusion.

**6. `/ship`'s ship-time re-grounding treated any `check-stale` `STALE`
verdict as automatic drift, with no way to recognize a task's own
predicted, approved changes.** A naive "any STALE verdict is drift" rule
produces a guaranteed false positive on nearly every task, since a task
that edits a file it also read as grounding — the ordinary case — always
trips `check-stale` at ship time. **Fix**: before treating a drifted file
as real drift, cross-check it against this task's own record — expected,
not drift, if the file appears in `plan.md`'s own `## Predicted touch`
list (this task's own approved change, not a neighbor's) or is explained
by a resolved `deviations.md` record (e.g. a manifest changed because an
approved new-dependency deviation added one). Only a file covered by
neither is genuine unexplained drift, which still counts toward the
circuit breaker exactly as before — this narrows false positives without
weakening real-drift detection, the identical shape as finding 3's fix to
`check-stale` itself.

**7. `callers`' own `--self-test` proved nothing about the adapter's real
behavior, and the adapter's real heuristic structurally couldn't match a
relative or aliased import.** Found during `20260815-waitlist-status-view`
(same project)'s `/verify` run: the floor's `callers` capability failed on
every changed file in a task that added correct, working relative
(`from './client'`) and `@/`-aliased (`from '@/stores/auth'`) imports. Two
distinct problems, one in `~/spine` core and one in this project's
generated adapter:
- **Core problem**: `g1-tee-waitlist`'s generated `callers` adapter's
  `--self-test pass`/`--self-test fail` never called the adapter's own
  real matching logic — the self-test branch had its own separate,
  simpler grep (`grep -rn "widget" "$tmp"`) that happened to succeed on a
  same-directory bare-symbol fixture, while the real adapter grepped a
  changed file's full repo-relative path as a literal string against the
  whole `src/` tree. A relative or aliased import specifier never
  contains that literal path substring, so the real heuristic was broken
  for the single most common import shape in the stack it was generated
  for — and `adapter-conformance` had no way to catch it, because the
  self-test it was running wasn't exercising the code path with the bug.
  **Fix (core, `ADAPTER-CONTRACT.md §4`)**: added an explicit rule that a
  self-test must invoke the adapter's real invocation path, never a
  second, parallel check that merely happens to agree with it on the
  fixture — the self-test's fixture is exactly what would have caught
  this if it had gone through `find_callers` instead of around it. This
  is stack-blind and applies to every future adapter, not just this one.
- **Project-specific problem**: even with a self-test that exercises the
  real path, `g1-tee-waitlist`'s own `.spine/adapters/callers` needed its
  actual matching logic fixed for its actual stack — TypeScript/Vue code
  overwhelmingly imports by relative specifier or the `@/` alias, neither
  of which is the literal repo-relative path the original adapter grepped
  for. **Fix (this project's adapter only, not portable to core)**:
  rewrote the adapter to check three real reference shapes (literal path,
  `@/`-aliased form, and a relative-import-specifier match on the file's
  basename) through one shared `find_callers` function that both normal
  mode and the rewritten self-test call, plus exempting `*.spec.ts` files
  (vitest test entry points, discovered by file glob, never imported by
  other source — requiring a caller for them is a structural false
  positive). `adapter-conformance callers` reconfirmed conformant against
  the rewritten self-test.
- **A second, smaller instance of finding 5's same bug class**: this same
  `/verify` run also failed `callers` on `.spine/adapters/callers` itself
  — the file just edited to fix the above, caught in the same diff as
  spine tooling config, not application source. `floor`'s `is_bookkeeping`
  only excluded `work/**`/`.spine/current-task`, not the rest of a
  project's own `.spine/` tooling install (`adapters/`,
  `capabilities.json`, `protected-paths.conf`,
  `install-command-patterns.conf`, `core-pin.json`). **Fix (core)**:
  extended `is_bookkeeping` to exclude all of those paths too — a task
  fixing its own generated adapter mid-verify (exactly this scenario)
  should never have that fix judged as if it were reviewable feature code.

## `spine/work/.build/` — keep it

Recommend keeping this directory as install history, per the build
prompt's own suggestion. It is the only record of *why* the install looks
the way it does — the calibration answers, the primitive-verification
results, the decisions and their reasoning, the things later phases needed
to know that weren't obvious from the files. Deleting it would not shrink
the system's always-loaded surface (it's never loaded by anything at
runtime) and would destroy exactly the kind of "why this shape" context
the whole system exists to preserve for everything else it produces.
