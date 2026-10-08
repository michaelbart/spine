# Enforcement map

What spine actually enforces, versus what it only asks a model to do. Built by reading the hooks, scripts, skills, agents and rules in `core/` (worktree `improvement-plan`). Purpose: decide, rule by rule, whether to convert prose to a check, delete it, or label it advisory.

## 1. Mechanisms

**Hooks.** All three fire on PreToolUse for `Edit|Write|Bash` and are wired through `.claude/hook-guard`, which denies every write if the real hook is not installed on this machine (`core/templates/hook-guard`). Bash writes are found by `_bash-write-targets`, which says of itself "a gate, not a shell parser"; a target it cannot place is denied. `_path-route` resolves a path to its project root and relative path. Neither helper is a hook.

- `phase-gate`: denies writes outside `work/<task-id>/` while `work/<task-id>/state` reads `research` or `plan`. It does nothing when there is no `.spine/current-task`, no `state` file, or any other phase. The model itself writes `state` (via `set-state`, or a direct edit, which the gate allows because the file is inside `work/<task-id>/`).
- `path-escalate`: denies writes to globs in `.spine/protected-paths.conf` unless `work/<task-id>/class` reads `2`; `#migration` globs are denied unless class is exactly 2. No active task means class 0. The model writes the `class` file.
- `dep-gate`: returns `ask` (a human prompt, which blocks headless) for writes to `#manifest` paths and for Bash commands matching `.spine/install-command-patterns.conf`. No patterns file means the command check is a no-op.

**Scripts that gate or check.** "Called by" says what makes it run. In every case a skill tells the model to run it; no hook or CI step does.

| Script | What it does | Blocks? |
|---|---|---|
| `floor` | Runs the project's adapters: typecheck, lint, then (class 1+) test, secret-scan, dep-diff, clone-scan, callers, mutate, smoke. Class 0 runs typecheck and lint only. | Exit 1 on first failing adapter. Unavailable adapters are recorded as degraded, never failed (except Class 2 smoke, with an M0 waiver). Called by task §1 (class 0), verify §1, ship §0. |
| `check-stale` | Compares `research.md` header (sha, files, cited decisions) to current state; marks the doc stale in place. | Exit 1 on drift, 2 on a bad header. Called by task §3, ship §0. |
| `verdict-filter` | Drops adversary verdicts with malformed or unresolvable evidence; rejects an empty `attacked` list. | Drops silently by design; fails only on unreadable input. Called by verify §3, design §5. |
| `content-sources-check` | For UI plans, requires every content block in `## Content sources` to cite a real tracked file or a human answer; `source: none` fails. | Exit 1. Called by task §3. |
| `design-gate` | Checks M0 targets, 13 capability statuses, six decision categories covered, at most 12 adopted decisions. | Exit 1. Called by design §7. |
| `adversary-cache-tier` | Computes REUSED / FOCUSED / FULL for adversary re-runs from content hashes. | Prints an answer only. Called by verify §3. |
| `profile-check` | Validates `.spine/profile.json`: unknown keys rejected, adversaries 1-2, class-0 limits capped (10 files, 100 lines). | Exit 1. Called by `setup --check` (task step 0). |
| `adapter-conformance` | Runs each adapter's pass/fail/cold self-tests and a shell-strictness lint. | Exit non-zero. Called by design §2, bootstrap, ship §3b. |
| `conformance` | Scores plan `## Predicted touch` against the real diff. | Never blocks (own header). Called by verify §4. |
| `ui-touch` | Reports whether the diff touches declared UI paths. | Informational (own header). Called by verify §1c. |
| `map-age`, `next-milestone-task`, `wayfinder-frontier`, `decision-index --check`, `decision-hash`, `gap-age` | Compute a status word or hash; the skill acts on it. | Never fail the run (exit 0 status) or are lookups. |
| `set-state` | Writes `state`, logs an event. Checks phase name and that the folder exists; does not check that the previous phase finished or that a plan was approved. | Bad name only. |
| `setup` | Writes machine-local symlinks and `hook-guard`; `--check` reports stale guard and invalid profile. | `--check` exit 1 on bad profile only. |
| `touchpoint-lint`, `repo-lint`, `core-selftest` | Static checks of skill text, README table, symlinks, citations, size budget; selftest runs the lot. | Guard the spine repo itself. Nothing runs them in a user project. |
| `spine-event`, `spine-stats`, `q`, `issue-ledger` | Logging, reporting, wrappers. `spine-stats` says "It never gates anything and nothing consults it." | No. |

**Not mechanisms.** Agent frontmatter (`disallowedTools`, `isolation: worktree`) and skill frontmatter (`disable-model-invocation: true`) are enforced by the Claude Code harness, not by spine code. They are rated `hook` below and labelled "harness".

## 2. Rule table

Strength: `hook` = a tool call is blocked or asked mechanically; `script` = a script computes or refuses, a model must be told to run it and act; `lint/test` = only static check of the text; `prose` = instruction only (advisory).

### Classify (`intake`, `task` §1)

| Rule | Stated in | Enforced by | Strength |
|---|---|---|---|
| Questions and exploratory spikes are not tasks; answer them outside the pipeline | intake §2 | Instruction text only | prose |
| Class is proposed with evidence and the human confirms it, never the model alone | task §1; intake §5-6 | Instruction text; the `class` file is written by the model, nothing records the confirmation | prose |
| Class 2 entry needs explicit human confirmation, never inferred | task §1 | Instruction text; `path-escalate` trusts whatever `class` file contains | prose |
| Class-2 signals (protected path, auth, migration, contract, 2+ systems) force Class 2 | intake §5 | Instruction text; only the protected-path part is backstopped later at write time by `path-escalate` | prose |
| Class 0 only if 2 files or fewer, about 15 lines, no new public symbol, no protected path | task §1; intake §5 | Instruction text; `profile-check` only caps the profile's thresholds; `path-escalate` catches the protected-path case | prose |
| A profile cannot disable the floor, protected-path hook, or autonomy cap; adversaries at least 1 | profile-check header; task §0 | `profile-check` rejects unknown keys and out-of-range values (mechanical); `setup --check` runs it (instruction) | script |
| Autonomy: Class 2 always `guided`; `auto` only at Class 1; capped by profile ceiling | task §0, §1, §3; intake §5 | Instruction text; `profile-check` validates only the ceiling's allowed values | prose |
| Run `setup --check` before classifying; invalid profile stops the task | task step 0 | `setup --check` exit 1 (mechanical); the model must run it and stop (instruction); a missing or stale guard only warns | script |
| Missing hook install denies every Edit, Write and Bash | bootstrap §4; `hook-guard` | `core/templates/hook-guard` exits 2 when `.claude/hooks/<name>` is missing | hook |
| Trivial edits still run `floor 0` (types and lint) and report degraded | task §1 | `floor` computes the result and logs `floor0`; the model must be told to run it | script |
| Refresh `docs/map.md` automatically when stale or empty | task §1 | `map-age` prints fresh/stale/empty/missing; the model must run it and spawn the refresh | script |
| Abandoning a task needs a stated reason | task Resuming | `set-state ... abandoned` refuses an empty reason; model must use it | script |
| Do not silently drop an active task; ask resume or abandon | task Resuming | Instruction text | prose |
| New milestone id: stop and confirm; review open Known gaps of shipped milestones | task header | Instruction text | prose |
| Bare `/task` proposes the next queued milestone task, never skips ahead | task header | `next-milestone-task` computes the answer; acting on it is instruction | script |
| Ticket branch name must contain `{ticket}`; create or reuse branch | task §1 | Instruction text | prose |
| `/verify`, `/ship`, `/task` can only be started by a human typing them | task §5; skill frontmatter | `disable-model-invocation: true` (harness). `auto` mode and `/intake` tell the model to "follow ... inline", which bypasses it | hook |

### Research (`task` §2, `researcher`)

| Rule | Stated in | Enforced by | Strength |
|---|---|---|---|
| Only `work/<task-id>/` is writable during research and plan | task state list; phase-gate header | `phase-gate` hook, Edit, Write and Bash. Inactive if `state` is absent or the model rewrites it | hook |
| Researcher cannot edit or write files | researcher.md frontmatter | `disallowedTools: Edit, Write` (harness); Bash is still allowed, and `phase-gate` is the only block on a Bash write | hook |
| Reply is the whole `research.md` with an exact machine header | researcher.md | `check-stale` parses the header and exits 2 if absent; only when someone runs it | script |
| Every claim traces to a file actually read; cite nothing unopened | researcher.md SHA-grounding | Instruction text; `check-stale` detects drift in listed files, not whether claims are true | prose |
| Decision citations use `decision-hash`, never a hand-written hash | researcher.md | `decision-hash` computes; `check-stale` re-checks the hash later | script |
| Class 1 is research-lite, Class 2 full | task §2 | Instruction text | prose |
| Bug fixes: reproduce first, then trace to root cause | researcher.md | Instruction text | prose |
| Config-dependent behavior: read the real setting, do not infer | researcher.md | Instruction text | prose |
| UI tasks: research must cite the screen JSON and screenshots | task §2; researcher.md | Instruction text; `check-stale` later notices if a cited mockup changes | prose |
| Discovery greps must offer a broader search before concluding | rules/search.md | Instruction text | prose |

### Plan (`task` §3)

| Rule | Stated in | Enforced by | Strength |
|---|---|---|---|
| If research is stale, redo it; do not ask the human | task §3 | `check-stale` marks the doc stale and exits 1 (mechanical); the model must run it and act | script |
| Own `milestone.md` bookkeeping edit is not drift | task §3; ship §0 | Instruction text; model compares `git diff` by hand | prose |
| A script that could not run is a tooling gap, never a silent pass | task header; verify header | Instruction text (notes.md line) | prose |
| `plan.md` follows the template and stays within 200 lines | task §3 | Instruction text; no script counts lines | prose |
| A plan touching a protected path escalates to Class 2 and forces `guided` | task §3 | Instruction text; `path-escalate` blocks the actual write later | prose |
| UI plans: every on-screen string or number has a named source; `source: none` stops and asks | task §3; rules/ui-design-system | `content-sources-check` exit 1 (mechanical); model must run it and not present the plan | script |
| Guided: present the plan, stop, wait for explicit approval | task §3 | Instruction text; no hook or script reads `approval.json` or checks it before `implement` | prose |
| Record `approval.json` with the approver identity | task §3 | Instruction text; nothing reads the file | prose |
| `auto` skips plan approval; the plan is reviewed in the PR | task §3 | Instruction text | prose |
| Plan names decisions it grounds on and known gaps it resolves | task §3 | Instruction text | prose |

### Implement (`task` §4, rules)

| Rule | Stated in | Enforced by | Strength |
|---|---|---|---|
| Edits to protected paths are blocked unless the task is Class 2 | task §1; path-escalate header | `path-escalate` hook | hook |
| Migration paths are denied unless class is exactly 2 | path-escalate header; rules/migrations | `path-escalate` hook (`#migration` tag) | hook |
| A Bash write whose target cannot be resolved is denied | path-escalate, phase-gate headers | `_bash-write-targets` UNRESOLVED branch, both hooks | hook |
| Package manifest edits need the human | task §4 halt list | `dep-gate` hook returns `ask` for `#manifest` paths | hook |
| Dependency installs need the human | task §4 halt list | `dep-gate` hook, only if `install-command-patterns.conf` exists and matches | hook |
| Deviations are matched to decide-alone / record-and-proceed / halt; halt means stop and ask | task §4 | Instruction text; hooks cover only protected and manifest files | prose |
| Record-and-proceed deviations go in `deviations.md`; tooling fixes go in `SETUP:` notes | task §4 | Instruction text | prose |
| Third deviation triggers the circuit breaker: stash, state back to `research`, redo research | task §4 | Instruction text; nothing counts records; `set-state` only writes the word | prose |
| A deviation citing a charter line ends in amend or reaffirm | task §4 | Instruction text | prose |
| Do not leave `plan` until approved | task §3-4 | Instruction text; `set-state` accepts any phase at any time | prose |
| Migrations: expand then contract in separate tasks; every migration ships a rollback | rules/migrations | Rollback is exercised by the `migrate-rehearse` adapter via `floor` (Class 2, diff touches a migrations dir, adapter implemented; otherwise degraded). The separate-task rule is unverified: the rule says the floor flags a co-commit, but `floor` has no such check | script |
| Auth changes: ground on the auth-model decision; flag a missing protected glob | rules/auth.md | Instruction text; the rule loads on file read | prose |
| UI: read tokens, components, screen spec and screenshot before writing; use declared tokens | rules/ui-design-system | Instruction text; `ui-conformance` later checks only that declared components rendered | prose |
| Undefined visual systems or missing tokens are a halt-tier deviation | rules/ui-design-system | Instruction text | prose |

### Verify (`verify`, `falsifier`, `security`, `ui-fidelity`)

| Rule | Stated in | Enforced by | Strength |
|---|---|---|---|
| Deterministic floor must pass for the class | verify §1 | `floor` exit code (mechanical); adapters are project-owned; missing adapters are degraded, not failed | script |
| A floor that could not run is FAIL, never PASS | verify header | Instruction text | prose |
| Skip adversaries on a diff the floor already rejected | verify §1 | Instruction text | prose |
| Falsifier always runs; profile cannot set adversaries to 0 | verify §2; profile-check | `profile-check` rejects 0 (mechanical); running the falsifier is instruction | script |
| Class 2 always runs both adversaries | verify §2 | Instruction text | prose |
| Adversaries are fresh subagents given only plan, diff and artifacts, no narrative | verify §3 | Instruction text; only falsifier has `isolation: worktree` (harness) | prose |
| Re-run tier REUSED / FOCUSED / FULL from content hashes and severity | verify §3 | `adversary-cache-tier` computes; the model must call it and obey | script |
| Adversary over budget is cancelled and not counted as a pass | verify §3 | Instruction text | prose |
| Read only the filtered verdict; evidence-free or empty-`attacked` verdicts are dropped | verify §3 | `verdict-filter` drops mechanically; model must run it and read the filtered file | script |
| Do not edit the tree while an adversary is running | verify §3 | Instruction text | prose |
| Security and ui-fidelity agents are read-only; ui-fidelity has no shell | security.md, ui-fidelity.md | `disallowedTools` (harness) | hook |
| Falsifier runs a stub-out probe; unrunnable probe is recorded as a gap | falsifier.md | Instruction text; no check that the probe ran | prose |
| Secret scan on adversary evidence; a secret fails verify | verify §3.5 | `secret-scan` adapter exit code (mechanical, project-owned); model must run it | script |
| UI render / conformance / fidelity run only if the diff touches a declared UI path | verify §1c-1e | `ui-touch` computes the fact; running the adapters is instruction | script |
| Plan-vs-diff score is informational; unlogged deviation noted | verify §4 | `conformance` never blocks; the note is instruction | script |
| `verify.md` quotes script output; write `disposition` back into verdicts | verify §5 | Instruction text | prose |

### Ship (`ship`)

| Rule | Stated in | Enforced by | Strength |
|---|---|---|---|
| Re-check grounding at ship; unexplained drift is a halt deviation, back to research | ship §0 | `check-stale` exit 1 (mechanical); the four-way "expected drift" cross-check is by hand | script |
| Rebase, then re-run the floor | ship §0 | `floor` (mechanical); running it is instruction | script |
| Merge gate: `verify.md` records PASS and zero open deviations, else stop | ship §1 | Instruction text; no script reads `verify.md` or counts `Status: open` | prose |
| `--bypass` only for emergencies; log it in notes, briefing, event, trailer | ship §1, §5 | Instruction text; `spine-event bypass` only logs | prose |
| Only the human can start `/ship` | task §5; ship frontmatter | `disable-model-invocation` (harness); `auto` follows it inline | hook |
| Cited decisions get implementing paths and `implemented` status; index regenerated | ship §2 | `decision-index` regenerates and `--check` detects staleness; the edits are by hand | script |
| Unfixed adversary findings are triaged; absent `disposition` counts as not fixed | ship §3a | Instruction text | prose |
| Unfixed `ui-fidelity` findings always need a human answer, even in `auto` | ship §3a | Instruction text | prose |
| Final milestone task: re-check the done-definition live; unmeasured means "unverified" | ship §3b | `adapter-conformance --all` exit code (mechanical); reporting is instruction | script |
| Briefing within 60 lines and about 600 words; PR text sourced from artifacts only | ship §4, §4a | Instruction text | prose |
| Commit carries `Spine-Task:` or `Spine-Bypass:` trailer | ship §5 | Instruction text; no commit-msg hook | prose |
| Guided tasks are not pushed; auto tasks get a draft PR and are never merged | ship §5 | Instruction text; `open-pr` is a project adapter, no hook blocks `git push` | prose |
| Set `done` only after the commit landed; clear `current-task` | ship §6 | Instruction text; `set-state` does not check git | prose |

### Setup and other skills

| Rule | Stated in | Enforced by | Strength |
|---|---|---|---|
| Design handoff refused until M0 targets, 13 capability statuses, six categories, decision cap hold | design §7 | `design-gate` exit 1 (mechanical); model must run it and not hand off | script |
| Human confirms each decision; none is `adopted` before design review | design §1, §6 | Instruction text | prose |
| Design review verdicts must cite real decisions with verbatim quotes | design §5; verdict-filter | `verdict-filter` decision-evidence check | script |
| Each kept design finding is revised or recorded as an override | design §6 | Instruction text | prose |
| A capability is `implemented` only after `adapter-conformance` passes | bootstrap §4; design §2 | `adapter-conformance` (mechanical); `floor` does not re-run it | script |
| `CLAUDE.md` spine block stays within 60 lines | bootstrap §5; ratchet | Instruction text | prose |
| Ratchet: a new rule must be a check, test, or protected path first; additions pay a deletion | ratchet | Instruction text | prose |
| Autopilot never pushes, refuses to take over a human task, stops after two circuit-breaker resets | autopilot §1, §4, §5 | Instruction text | prose |
| Roadmap: every known gap, issue and decision follow-on is placed or explicitly declined | roadmap §4 | Instruction text | prose |
| Wayfinder frontier and blocked state are derived, never stored | wayfinder | `wayfinder-frontier` computes; model must run it | script |
| Skill text follows the touchpoint block format; markers are never printed | human-touchpoint.md; rules/touchpoint-output | `touchpoint-lint` in selftest (format only); "never print markers" is instruction | lint/test |
| Skill line caps, 18 commands, 25 scripts, 3150 script code lines; README lists every command | budget.json | `repo-lint` in selftest, spine checkout only | lint/test |

## 3. Findings

**Counts** (92 rows): `prose` 53 (58%), `script` 26 (28%), `hook` 11 (12%), `lint/test` 2 (2%). Of the 11 `hook` rows, 4 are harness frontmatter (agent tool lists, `disable-model-invocation`), and two of those (human-only `/verify` and `/ship`) are bypassed by `auto` mode; only 7 are spine PreToolUse hooks or the guard. Every `script` row still depends on a model choosing to run the script.

**Ten most important prose-only rules**

1. Plan approval before implementation (task §3). Convertible: have `set-state implement` refuse unless `approval.json` exists and names a git identity. Today a model can write `state` itself and nothing notices.
2. Ship merge gate: floor PASS and no open deviations (ship §1). Convertible: a 10-line script that reads `verify.md` and counts `^- Status: open`, run by ship and refusing to continue on exit 1. The skill already describes the exact `grep -c`.
3. Class is human-confirmed, and Class 2 is never inferred (task §1). Partly convertible: have the skill write `class` only via a script that also records an `AskUserQuestion` answer; otherwise label advisory, since `path-escalate` trusts the file.
4. Circuit breaker at the third deviation (task §4). Convertible: `set-state` or a hook counts `^- Tier:` records in `deviations.md` and denies `implement` at three.
5. Halt-tier deviations stop and ask (task §4). Label advisory beyond the protected-path and manifest cases the hooks already cover; a general check is not possible.
6. Commit carries `Spine-Task:` or `Spine-Bypass:` (ship §5). Convertible: a `commit-msg` git hook, or have ship commit through a script.
7. Do not push or merge from guided or autopilot runs (ship §5, autopilot §4). Convertible: a `dep-gate`-style PreToolUse check on `git push` and `gh pr merge`, honoring `autonomy`.
8. `plan.md` 200-line cap and briefing 60-line cap (task §3, ship §4). Convertible: add to a single `lint-artifacts` script called by verify; or delete the caps and keep the writing mandate.
9. Adversaries are given no implementation narrative and run within a time budget (verify §3). Not convertible (the dispatch text is the model's); label advisory. Only the budget could be a harness setting.
10. Researcher claims trace to files actually read (researcher.md). Not convertible; keep as a requirement but label advisory. `check-stale` already covers the part that can be checked (drift).

Also worth deleting or labelling: `rules/search.md` (pure advice), and the migrations rule's claim that "the deterministic floor flags" a destructive-schema plus app-code co-commit, which `floor` does not do.

**Where the README overstates**

- "Every gate is a real script or a real hook, not a prompt asking an agent to please be careful." 53 of the 92 rules above are prose. Even for the gates that do have a script, a skill must tell the model to run it; no hook or CI step runs `floor`, `check-stale`, `verdict-filter` or `design-gate`. The three real hooks cover file writes and package installs only. Plan approval, the merge gate, the circuit breaker, class confirmation, trailers and "never push" have no mechanism.
- "Everything a model does is steerable by a good-enough prompt; everything a hook or script does isn't." The hooks read `state` and `class` files that the model writes, and `phase-gate` allows writes inside `work/<task-id>/`, so the model can move itself out of the gated phase. `disable-model-invocation` also stops applying where `auto` mode and `/intake` say to "follow ... inline".
- `conformance`, `ui-touch`, `spine-stats` and the `ui-fidelity` review are described in skills as checks but do not block anything by design.
