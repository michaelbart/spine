# Spine improvement plan

Status: proposed 2026-10-08. Nothing below is done unless its checkbox says so.
Update this file in the same commit that completes an item (tick the box, add
the commit SHA). A future session should start here, read "Ground rules" and
"Dependencies", then pick the first unticked item whose prerequisites are done.

## Why this plan exists

A review of `docs/tradeoffs.md`, the README, the three largest skills, and real
data from installed projects (turnpilot: 92 task folders / 364 events; tgml: 80
task folders; turnpilot session transcripts) produced four conclusions:

1. **Spine has no evidence it pays for itself.** The reporting layer was deleted
   (`1524cd8`, -3.2k lines). `docs/tradeoffs.md` says "no built-in
   instrumentation". The overhead estimate (2-4x) is a guess.
2. **The accretion criticism is partly valid.** 20 skill dirs (19 commands; an earlier count of 22 included four dangling symlinks), 5.6k lines of
   skill prose (`task` 911, `ship` 782, `verify` 653), ~8k lines of scripts and
   hooks, a 3k-line selftest (68s), 117 commits in 8 weeks, six audit rounds in
   one day. Evidence of drift: four committed symlinks point at skills deleted
   in `1524cd8`; `/ticket` is missing from the README; the second-approver rule
   existed for weeks and was self-overridden 78 of 82 times. The part of the
   criticism that does not hold: the daily interface is `/task`, `/verify`,
   `/ship`, `/spine` and three touchpoints.
3. **There is measurable friction.** turnpilot logged 301 hook interventions
   across 11 shipped tasks (145 `dep-gate` asks paired with 145 `path-escalate`
   blocks, all on Bash; one task had 52). The log does not record why.
   Roughly two-thirds of turnpilot tasks are Class 2 and nearly all `guided`,
   so classification may not discriminate.
4. **Some tradeoffs-doc entries are real gaps with known fixes**, some are
   genuine tradeoffs, and many are inherent to any process.

## Ground rules

- **Measure before pruning.** Workstream B (measurement) precedes D (pruning)
  and the data-dependent decisions in C and E. Only workstream A and the
  logging pieces of B/C may be done first.
- **Complexity budget.** From D1 onward, adding a command, mode, skill line or
  script must be paid for by removing something, or the budget file is raised
  with a written justification in the commit.
- **New measurement never gates.** Nothing may consult stats to allow or block
  work. Same rule the event log already follows.
- **Prefer a script or hook to a prose rule.** If a rule cannot be mechanized,
  label it advisory and say so.
- **Every code change gets a `core-selftest` case, and a negative control**
  (revert the fix, watch the new test fail), per the standing lesson in
  `docs/audit-2026-08-22-round3.md`.
- **No per-project data leaves the machine.** The analyzer reads transcripts
  and task folders locally and emits numbers only, never content.

## Owner decisions (2026-10-08)

1. **Team coordination and cross-repo support are removed** (workstream G).
   The owner is a solo engineer; this machinery serves no current user.
2. **No `lite` lane for now.** Compare spine-attributed vs non-spine work in
   existing sessions first; B8's lane is a contingency only if that comparison
   is too confounded.
3. **Class 2 for no-runtime projects (E3) is wanted but low priority**, to be
   done only if it stays as small as the M0 waiver was.
4. **Baseline projects: turnpilot first (largest spine use), tgml second.**

## Dependencies

```
A1,A2,A4,A5 ─────────────────────────────────────────────────────────┐
G (remove team/cross-repo) ──> A3 (work/ story), A6 ──> C2, D4, D8     │
B1-B3 (schema + events) ──> B4-B7 (analyzer, baseline) ──> D, C3, D7,  │
                                                  E-decisions ──> F ◄──┘
C1 (hook reason logging) ──> C2 (fix, on the simplified hooks)
```

A1/A2/A4/A5, G, B1-B3 and C1 can start immediately and in parallel. G runs
early despite its letter: it deletes code that C2, D4 and D8 would otherwise
have to modify or split, and it simplifies the hook layer the friction fix
targets. F is last because pruning changes what the docs should say.

---

## A. Hygiene and correctness

Small, independent, no data needed.

- [x] **A1. Remove dangling skill symlinks.** `.claude/skills/{costs,task-report,tasks,visualize}`
  are committed but their targets were deleted in `1524cd8`.
  Add a `core-selftest` case: every symlink under `.claude/skills`,
  `.claude/agents`, `.claude/rules` resolves, and `setup` never creates a
  dangling one. Negative control: re-add one, test fails.
- [x] **A2. Make the README command table complete and checked.** Add
  `/ticket` and `/autopilot` (both were missing). Add a selftest/lint (`readme-commands-check`) that every
  `core/skills/*/SKILL.md` with `disable-model-invocation: true` appears in the
  README table, and every README command has a skill. `security-checklist` is
  explicitly exempt (agent-preloaded background, not a command).
- [ ] **A3. Reconcile the `work/` sharing story (after G, which decides
  whether task state stays committed at all).** Current facts to confirm by
  reading code: `setup` (`core/scripts/setup` ~L360-370) now leaves task state,
  owner and claims committed while ignoring artifacts; `registry-sync`'s header
  still says whole task folders are committed and pushed; README "Working with
  other engineers" says `/task` commits and pushes its task folder; the
  tradeoffs worktree item assumes registry pushes to the checked-out branch.
  Deliverable: one accurate description of what is shared, what is local, and
  what each of README / registry-sync header / tradeoffs / task skill says
  about it, with every copy of the story corrected. Then re-evaluate whether
  the worktree-isolation limit still exists as written.
- [x] **A4. Remove the dead "parallel work streams" knob.** (Done: the field does not exist anywhere in `core/`, installed projects or git history; the Deferred row was simply wrong and is deleted.) `tradeoffs.md`
  Deferred table says "Calibration has the field; it's hardcoded off". Find the
  field (grep found no match in `core/`, so it may live in generated
  calibration output or `~/.spine`), delete it and the Deferred row.
- [x] **A5. Stale-reference sweep.** (Done 2026-10-08: no change needed; remaining matches are plain-English "ledger", the live `issue-ledger` script, and a historical selftest comment.) After A1-A4, grep live docs and skills for
  removed features (ledger, dashboard, costs, visualize, task-report,
  second-approver). Currently clean except intended mentions
  (`tradeoffs.md` second-approver history, `task/SKILL.md:755` rationale,
  `spine/SKILL.md:134` "ledger" meaning the known-issues list: reword to avoid
  confusion).
- [x] **A6. `doc-refs` check (in `core/scripts/repo-lint`, with A1/A2's checks).** Dead citations were a finding in audit
  rounds 1, 3, 4, 5 and 6. Add a script that scans `*.md`, hooks and scripts for
  `core/...` paths, `§N` section references and `docs/...` links and verifies
  the target exists (join wrapped lines before matching, per round 4's lesson).
  Run from `core-selftest`. Audit reports are exempt (they quote dead refs).

## B. Measurement (`spine-stats`)

Goal: answer "does spine pay for itself, and where does it cost most?" with
numbers from real projects. Read-only, deletable, never on the gate path.

### B1. Event schema

- [ ] Write `docs/event-schema.md`: every event, every field, a `v` (schema
  version) field on each line, rules (append-only, never read for decisions,
  never fails a task, no secrets).

### B2. Richer hook events

- [ ] `hook-block` / `hook-ask` record: `rule` (a stable id such as
  `unresolved-bash-target`, `protected-path`, `manifest`, `install-command`,
  `phase`), `target` (repo-relative path when resolved), `cmd0` (first token of
  a Bash command only), and `cmd_sha` (short hash of the full command, to
  group repeats). **Never store the full command** (may contain secrets).
- [ ] Selftest asserts the fields and asserts no full command appears.

### B3. New lifecycle events

Add through `spine-event` calls in skills (or scripts where possible):

- [ ] `task-start` (class unknown yet), `class-set` (exists), `phase`
  (research/plan/implement/verify/ship, entered), `plan-approved`,
  `deviation` (kind=real|setup, tier), `verify-round` (n, floor=pass|fail|degraded,
  falsifier and security finding counts by severity), `finding-disposition`
  (fixed|carried|declined, agent, severity), `abandoned` (see E2),
  `shipped` (exists; extend with `tokens_*` rollup, see B4).
- [ ] `lane=full|lite` field on `task-start` (used only by B8).
- [ ] Skills call `spine-event`; none of these may add a stop or a prompt.

### B4. The analyzer

- [ ] `core/scripts/spine-stats` (bash + jq, matching repo idiom; no new
  runtime dependency). Subcommands: `summary`, `task <id>`, `commands`,
  `friction`, `adversaries`, `floor`, `--since <date>`, `--project <path>`,
  `--json`.
- **Inputs:**
  1. `.spine/events.jsonl`
  2. Claude transcripts `~/.claude/projects/<encoded-cwd>/*.jsonl` and
     `<session>/subagents/*.jsonl` + `*.meta.json` (`agentType`). Per assistant
     message: `message.usage` (`input_tokens`, `cache_creation_input_tokens`,
     `cache_read_input_tokens`, `output_tokens`), `attributionSkill`,
     `isSidechain`, `timestamp`.
  3. `work/<task>/` artifacts (class, autonomy, deviations, verify, briefing;
     word counts as a proxy for reading load).
  4. git / `gh` (commit times, merge time, reverts, follow-up commits).
- **Task attribution:** join transcript turns to a task by project cwd and the
  `[task-start, shipped]` window from events; subagent turns inherit the parent
  session's window. Where events are missing (history before 2026-10-06), fall
  back to task-folder mtimes and `plan-approved`/state file times, and mark
  the row `inferred`.
- **Survives transcript pruning:** at `/ship`, compute the task's token rollup
  and write it into the `shipped` event (`tokens_fresh_in`, `tokens_cache_read`,
  `tokens_out`, per phase and per agent type). The analyzer prefers the rollup.
- **Prices:** `core/data/prices.json`, versioned by date, user-overridable at
  `~/.spine/prices.json`. Cost is always shown with the price-table date.
- **Known uncertainty to resolve first:** how `attributionSkill` is assigned
  (every turn under a skill, or only the first). Verify empirically on a
  controlled session before trusting phase splits.

### B5. Metrics (definitions)

| Metric | Definition |
|---|---|
| Spine token share | tokens attributed to spine skills and subagents / all tokens in the same project and window |
| Tokens per task | by phase and by agent type (researcher, falsifier, security, ui-fidelity) |
| Cycle time | `task-start` to merge; split machine time vs wait-on-human (gap between assistant stop and next user message, at touchpoints) |
| Touchpoint load | count and minutes of human wait at classify / plan / briefing |
| First-pass verify rate | tasks whose first verify round is PASS |
| Verify rounds per task | count of `verify-round` |
| Late deviation rate | `deviation` events after `plan-approved` per task |
| Rework rate | share of shipped tasks with a follow-up commit within 14 days touching >=50% of the task's changed files (excluding docs-only and merge commits), plus reverts |
| Escaped defects | `known-issues.md` entries citing a shipped task |
| Adversary yield | findings with disposition `fixed` / findings raised, per agent and severity. An agent with ~0 yield over N tasks is cost without benefit |
| Floor-layer value | per floor layer: runs, failures, `degraded`. A layer that never fails across N tasks is a pruning candidate |
| Hook friction | interventions per shipped task, by `rule`; bypass count |
| Classification | class distribution; class-2 tasks with zero fixed findings and zero deviations (over-classification candidates) |
| Command census | invocations per command per 30/60 days (from `attributionSkill` and events) |
| Complexity | lines per SKILL.md, command count, script count, selftest runtime |

Rework and escaped-defect detection are heuristics. The report must say so and
show the matching commits so a human can judge each one.

### B6. Baseline report

- [ ] Run B4 over turnpilot (primary: ~92 task folders, event log since
  2026-10-06) and tgml (secondary: ~80 task folders, predates the event log so
  most rows are `inferred`). Commit the
  numbers (not content) as `docs/baseline-2026-10.md`. It must answer: spine
  token share; why `/ship` consumes more than `/task` (22M vs 12.6M fresh input
  tokens in the turnpilot spot-check, unverified); adversary yield; hook noise
  by rule; class distribution; command census.

### B7. Surfacing

- [ ] `/spine` idle report shows a 5-line summary from `spine-stats summary`
  (no new command). Silent if the analyzer or jq is missing.

### B8. Counterfactual (contingency: owner chose comparison-first)

Without a baseline, the numbers say what spine costs, not whether it earns it.
Step 1 (do this): compare spine-attributed work to non-spine work in the same
projects, already visible in transcripts and git. Step 2 (only if step 1 is too
confounded to be useful, e.g. non-spine work is a different kind of change):

- [ ] Optional `lite` lane for a random fraction of eligible Class 1
  tasks (implement + floor + one-line trace; no research/plan/adversary),
  fraction set in `profile.json` (`lite_lane_rate`, default 0). Compare rework
  and escaped-defect rates after N tasks. Solo sample sizes are small, so
  report directionally, with counts, never as significance.
- This adds a mode, so it carries an **expiry**: remove the lane and field
  after the sample target is reached or 90 days, whichever is first. The
  budget (D1) must be satisfied by removing something else first.
- Alternative with no new mode: compare spine-attributed vs non-spine work in
  the same projects (already visible in transcripts). Do this first; build the
  lane only if the comparison is too confounded to use.

### B9. Tests

- [ ] Synthetic transcript, subagent and event fixtures in `core-selftest`:
  token sums, phase split, task window join, pruned-transcript fallback to the
  rollup, price-table override, and a privacy test (output contains no prompt
  or code text).

---

## C. Hook friction

### C1. Find out why (needs B2)

- [ ] Ship B2, run normally for a stretch, then group `hook-*` events by
  `rule` / `cmd0` / `cmd_sha`.
- Hypothesis to confirm or reject: `path-escalate` denies and `dep-gate` asks
  for any Bash command whose write target `_bash-write-targets` cannot resolve
  (compound commands, pipes, `cd && ...`, package scripts), which is why the
  145 + 145 events are paired. Also check whether commands that write nothing
  are being flagged.

### C2. Reduce it without weakening it

Candidate fixes, to be chosen from C1's data:

- [ ] Read-only and known-safe command allowlist (`git status|diff|log`, `ls`,
  test/typecheck/lint runners per adapter) that skips write-target analysis.
- [ ] Better target resolution for common shapes in `_bash-write-targets`.
- [ ] One ask per command shape per task (cache the human's answer in
  `.spine/` for the task) so repeats do not re-prompt.
- [ ] One command produces one intervention: define precedence between
  `path-escalate` (block) and `dep-gate` (ask) so a single Bash call cannot
  trigger both.
- Constraint: a Bash write that genuinely cannot be resolved stays fail-closed.
  The fuzz suite (`core-selftest` fuzz section) must still pass, and new fuzz
  cases must cover every newly allowlisted shape.

### C3. Acceptance (set the number after B6)

- [ ] Interventions per shipped task reduced by a target agreed from the
  baseline (suggested: >=70% fewer), with zero new bypass cases in the fuzz
  suite and no loss of protected-path or manifest coverage.

---

## D. Complexity control

### D1. Budget

- [ ] `core/budget.json`: max lines per `SKILL.md`, max user-facing commands,
  max scripts, max selftest runtime. `core-selftest` fails when exceeded.
  Initial values = current values (freeze); tighten after D4.
- [ ] Rule: raising a cap requires the commit message to say what was removed
  or why that is impossible.

### D2. Command census and tiers (needs B4)

- [ ] From the census classify each command: **core** (used weekly),
  **occasional** (setup / planning), **dormant** (not used in 60 days).
  Candidates to review: `/prompts`, `/remap`, `/ratchet`, `/note-issue`,
  `/ticket`, `/autopilot`, `/wayfinder`, `/prototype`. (`/workspace` is
  removed by G, not judged here.)
- [ ] README command table regrouped into the same tiers, with a one-page
  "the 80% path" at the top (`/spine`, `/task`, `/verify`, `/ship`).

### D3. Demotion mechanism

- [ ] Decide how a dormant command leaves the default surface without being
  deleted: `core/skills-extra/` not wired by default, `setup --with-extras`
  to opt in, or a `tier: advanced` frontmatter field that `setup` and
  `/spine` honor. Inspect `core/scripts/setup` first for how skills are wired.
  Dormant for 2 consecutive censuses with no one asking for it -> delete.

### D4. Split the big skills (progressive disclosure)

- [ ] `task`, `ship`, `verify`: keep the common path in `SKILL.md`; move
  branch-specific material to `reference/*.md` files loaded only when a stated
  condition holds (Class 2, workspace/multi-repo, worktree, autopilot, UI
  fidelity, milestone bookkeeping). Target <=350 lines each for the main file.
- [ ] Each reference has an explicit trigger line in `SKILL.md` and a lint that
  every reference is reachable from a trigger and every trigger resolves.
- [ ] Verification: token count per `/task` invocation before/after (B4);
  `core-selftest` passes; replay one golden Class 1 and one Class 2 task on a
  fixture project and compare produced artifacts.
- Risk: the model skips a reference it should have loaded. Mitigation: the
  gates that matter are scripts and hooks, not prose; the lint above; the D5
  map shows which rules rely on prose.

### D5. Enforcement map

- [ ] `docs/enforcement-map.md`: for every "must / never / always" rule in
  skills and agents, the mechanism that enforces it (hook, script, selftest, or
  prose-only). For each prose-only rule choose: convert to a script check
  (`/ratchet` path), delete, or tag `[advisory]`. Rewrite the README "teeth"
  claim to match the map exactly.

### D6. Rationalize modes and knobs

- [ ] **Autonomy:** data so far: `guided` and `auto` are used heavily
  (turnpilot 59 guided / 5 auto; tgml 37 auto / 13 guided);
  `checkpointed` is ~absent from task folders (one event in turnpilot). Unless
  B6 shows real use, remove `checkpointed` everywhere (37 references across
  six skills, plus the `auto-checkpointed` profile value).
- [ ] **`/autopilot`:** keep, demote (D3) or remove based on the census. If
  kept, it is one experimental entry, not a mode threaded through other skills.
- [ ] **Profile and calibration fields:** list every field in
  `core/templates/profile.json`, `capabilities.json`, calibration output; delete
  any that nothing reads (extend `profile-check` to report unread fields).
- [ ] **Experimental flags** (`ui_fidelity_class1_optin`, worktree isolation):
  keep only those with recorded use.

### D7. Classification quality

- [ ] Investigate Class 2 share (turnpilot ~2/3). Suspect: `e7a0b55` made
  `/intake` count protected files a change *reads from*. Compare class
  distribution before and after that commit, and check turnpilot's broad
  protected-path globs (billing, deposits, invoices, db, jobs, auth) against
  the greenfield milestone it was building.
- [ ] Use the "class-2 with zero fixed findings and zero deviations" metric.
  If over-classification is confirmed: Class 2 triggers on *writes* to
  protected paths and reads become a note; or recalibrate protected-path
  conf at `/adopt`. Decide from data, record in a decision doc.

### D8. `/ship` cost

- [ ] From the B6 phase breakdown find where `/ship` spends tokens (re-grounding,
  decision distillation, milestone bookkeeping, briefing, PR description).
  Move deterministic parts to scripts; split the skill's "six jobs" into
  sequential steps that load only what they need.

### D9. Audit practice

- [ ] Stop writing round-numbered audit reports into `docs/`. Move
  `docs/audit-2026-08-22*.md` to `docs/history/`. Replace with continuous
  checks: fuzz suite, selftest, A6, budget, README check. Keep one
  `docs/history/README.md` index.

---

## E. Real gaps

### E1. Class 0 floor subset

- [ ] Class 0 runs the changed-file lint/type layer (the narrow fix
  `tradeoffs.md` already names). `floor --class 0` runs only those layers;
  degrades with a visible line when the adapter lacks them; result recorded in
  the trace line. Selftest per outcome (pass, fail, degraded).

### E2. Abandoning a task

- [ ] Design first, as a short decision record, then implement. Recommended
  shape: `/task --abandon <id> "<reason>"` (no new command). Terminal state
  `abandoned` with a required reason, `abandoned` event, final registry sync,
  `.spine/current-task` cleared if active, no further writes allowed by
  `phase-gate`.
- Milestone fork (the open design question): an abandoned member task makes its
  milestone "needs disposition". `next-milestone-task` reports the milestone as
  blocked-pending-disposition; `/spine` surfaces it; the human either adds a
  replacement task or descopes via `/roadmap` (recorded in the milestone's
  Known gaps). Recommend this over auto-replacing.
- [ ] Three mechanisms to update, each with a selftest and negative control:
  `next-milestone-task` completeness check, `claims-check` live-claims
  predicate, `current-task` clearing. Also `/spine` status and `phase-gate`.

### E3. Class 2 for projects with no runtime (LOW PRIORITY, do last in E)

- [ ] Wanted by the owner only if cheap; the M0 waiver (`d846fc7`) is the size
  to aim for. If the design grows past that, drop it. Spine itself is a
  candidate to dogfood. Mirror the M0 waiver: `floor` records
  `degraded:no-runtime` instead of failing only when the charter declares
  `Runtime: none`, `capabilities.json` marks smoke `not-applicable` with that
  reason, and the unit/integration test layer is `implemented`. One selftest
  per missing fact; reverting the waiver must fail them.

### E4. `ui-capture` / `ui-fidelity`

- [ ] Run end to end once on a real UI project with a real adapter; record the
  calibration results `docs/proposal-ui-fidelity.md` promised. Outcome is
  binary: document results and keep, or remove the agent, scripts, template
  and `ui-touch` content-path code. Do not leave "never run" in the repo.

### E5. Unlogged-deviation backstop

- [ ] The circuit breaker is an honor system. Add a verify-time warning (not a
  block): files changed outside the plan's declared file set, or fix commits
  during implement, with zero logged deviations -> "possible unlogged
  deviation", shown in `verify.md` and the briefing. Also cheap: aggregate
  setup events and deviation counts across tasks in `spine-stats` (closes the
  "no project-wide aggregate view" limit).

### E6. Plan-quality proxy (optional)

- [ ] Shape lint for `plan.md` acceptance checks: each names a command or
  observable plus an expected result, and at least one is a negative check.
  This catches empty plans, not wrong ones; say so in the plan template. Add a
  metric: tasks whose FAIL or late deviation traces to an acceptance gap.

### E7. Whole-tree floor

- [ ] Document a CI recipe for `floor --whole-tree` nightly. `/spine` shows the
  age of the last whole-tree result (`gap-age` style) when one is recorded.

### E8. Adversary independence (evaluate, do not assume)

- [ ] After B6 gives adversary yield: pilot routing falsifier/security to a
  different model than the implementer and compare yield. Keep only if yield
  rises enough to justify the cost.

### E9. Explicitly deferred (reviewed, not doing now)

Record in the Deferred table with a review date and a trigger:

- Reverting a shipped task: `git revert` is adequate for a solo engineer;
  revisit if a revert ever goes wrong.
- Conformance-gated pin bumps, immediate pin-bump notice, committed
  `.claude/hooks/` (`docs/proposal-committed-hooks.md`): moot once G removes
  the pin machinery; move the proposal to `docs/history/`. Trigger to revisit:
  a second engineer or second machine.
- Parallel / team-of-agents orchestration, concurrency lane: no demand.
- Multi-workspace repos: removed with G.

---

## G. Remove team and cross-repo support (decided; runs early)

Owner is solo; none of this serves a current user. Scope measured 2026-10-08:
~2.4k lines directly, workspace-aware branches in 12 skills, ~184 lines of
`core-selftest`, and `_workspace-route` (360 lines) which all three
enforcement hooks depend on. Removing it also deletes the code where six
rounds of hook-bypass bugs were found.

**Remove:**

- `/workspace` skill, `core/templates/workspace.json`, workspace mode in
  `setup`, `--root` / `--from-design` handling.
- Multi-repo routing: `core/hooks/_workspace-route`'s routing to member repos.
  Keep and re-home only what single-repo hooks need (`_wsr_collapse`,
  `_wsr_realpath`, glob matching) in a small helper, with their fuzz cases.
- Cross-engineer coordination: `registry-sync`, `propagate`, the
  cross-engineer parts of `claims-check`, `core/templates/claims.json`, the
  `owner` file, flag acknowledgment, git-identity use.
- Cross-repo contracts: `contract-touch`, `core/rules/contracts.md`, the
  `contract-check` capability in `core/ADAPTER-CONTRACT.md` and the floor
  layer that runs it, the contract sections in plan / verify / briefing /
  pr-description templates and in the falsifier agent.
- Core-pin machinery: `.spine/core-pin.json`, version-skew detection in
  `setup --check`, `/update --bump-pin`. Keep `/update` basic sync and the
  install-health part of `setup --check`.
- Workspace branches in: task, ship, verify, intake, note-issue, ticket, spine,
  autopilot, bootstrap, researcher, decision-index, check-stale, ui-touch.
- The matching `core-selftest` cases (record count before and after).
- Docs: tradeoffs "Working with other engineers", "Cross-repo work", pin rows
  of the Deferred table; move `docs/proposal-committed-hooks.md` to
  `docs/history/`.

**Decisions to confirm while doing it (none block starting):**

- [ ] **`claims-check` / worktree isolation.** Without teammates, claims only
  guard two tasks in one checkout (worktree isolation, "Extension D",
  experimental). Check the census: if worktree isolation is unused, remove it
  and `claims-check` entirely; if used, keep a local-only overlap check.
- [ ] **`work/` sharing.** With no other engineer, task state has no reader
  but the owner. Default: gitignore all of `work/` (as `1524cd8` began),
  keeping distilled `docs/decisions/` committed. Confirm before changing
  `setup`. This settles A3.
- [ ] **`bookmarks-workspace`** is the only installed workspace (4 tasks, in
  `/Users/michaelbart/bookmarks-workspace`). Confirm it is disposable or pin
  it to the pre-removal tag; do not leave it silently broken.

**Steps (each its own commit):**

1. [ ] Tag the current `main` as `pre-team-removal`.
2. [ ] Delete the skill, scripts, templates and rules with no hook coupling
   (`/workspace`, `propagate`, `registry-sync`, `contract-touch`,
   `contracts.md`, templates) and their selftest cases.
3. [ ] Simplify the hooks: replace `_workspace-route` with the small
   single-repo helper. **Own commit**, because these are the security layer.
   The fuzz suite for the kept functions must pass; each retained behavior
   keeps a negative control.
4. [ ] Strip workspace / claims / pin branches from the skills and scripts
   listed above.
5. [ ] Remove pin and skew code from `setup` / `update`.
6. [ ] Docs sweep (also satisfies part of F2 / F5).

**Acceptance:** `grep -rn` for `workspace_route`, `workspace.json`,
`claims-check`, `registry-sync`, `propagate`, `contract-touch`, `core-pin`
finds nothing outside `docs/history/`; `core-selftest` is green and its case
count and runtime are reported before/after; a single-repo golden replay
(one Class 1, one Class 2 task on a fixture project) produces the same
artifacts as before; line counts removed are recorded in the commit message.

---

## F. Documentation

Last, because D changes what is true.

- [ ] **F1. Split `docs/tradeoffs.md`** (500 lines, four genres) into: a
  one-page `tradeoffs.md` (what it costs, when it is the wrong tool, when to
  abandon it, what makes it obsolete, accepted limits) and `docs/design/`
  holding the rationale sections (human touchpoints, visual fidelity, M0 waiver,
  design stage, worktree/autopilot notes) plus the proposals.
- [ ] **F2. Apply the triage.** Each Known-limits entry becomes one of:
  - *Fixed* (E1, E2, E3, E5): remove, link the change.
  - *Tradeoff, measured*: keep with the metric that tracks it (adversaries
    don't gate -> adversary yield and shipped-with-high-finding count; plan
    backstop -> E6 proxy; honor-system deviations -> E5).
  - *Inherent*: collapse to a single bullet list with no prose: seeded data
    is not production data; concurrency out of scope; semantic collisions
    between tasks; charter rot; evidence checked for shape not truth; write
    hooks do not see every write shape; per-task floor sees only the diff
    (point at E7).
  - *Team / cross-repo*: removed by G; delete the sections, no replacement.
- [ ] **F3. Replace guesses with data.** "Token overhead" and "Human minutes"
  quote B6 numbers with a date and project count, or say "unmeasured".
- [ ] **F4. Review dates.** Each remaining accepted limit carries
  `reviewed: YYYY-MM` so stale claims are visible.
- [ ] **F5. README.** Command table in tiers (D2), the 80% path, accurate
  "teeth" claim (D5), work/ sharing as reconciled in A3.

---

## Definition of done for the whole plan

1. `spine-stats summary` runs on two real projects and the baseline is
   committed.
2. Hook interventions per shipped task are down by the agreed target with the
   fuzz suite green.
3. `core/budget.json` exists, is enforced, and caps are tighter than the
   2026-10-08 starting values (task 911 / ship 782 / verify 653 lines;
   target <=350 each for the main files).
4. Every user-invocable skill is in the README; no dangling symlinks; doc
   references are checked.
5. `tradeoffs.md` fits on a page and every remaining claim cites a metric or
   an accepted-limit review date.
6. Class 0 has a floor subset; abandoning a task works; ui-capture is either
   proven or gone.

7. Team, cross-repo and pin machinery are gone (G) and the hook layer is
   single-repo only.

## Open questions

Answered 2026-10-08: see "Owner decisions". Remaining, all inside G:
`claims-check` / worktree isolation fate, whether all of `work/` becomes
local-only, and what to do with `bookmarks-workspace`.
