# Phase C Handoff — Skills, Agents, Artifact Templates, CLAUDE.md, README

Date: 2026-08-08. Builder: Claude Code (Sonnet 5), interactive session (fresh session from Phase B per the build prompt's session-boundary discipline).

Read order for the resuming session: the build prompt, `phase-A-handoff.md`, `phase-B-handoff.md`, this file, then the state of `/Users/michaelbart/spine` and `/Users/michaelbart/horizon` on disk.

## 1. What was built

**`spine/core/templates/`** — every artifact format the skills below produce, each a fill-in-the-blank file with an explanatory HTML comment for the human/model filling it in, never itself naming a language, framework, or tool:

| File | Status |
|---|---|
| `charter.md` | New. `DRAFT`/`CONFIRMED` marker convention; ≤100-line discipline stated in-file. |
| `map.md` | New. `<!-- spine:map sha:... generated:... -->` header (not machine-parsed by any script — `check-stale` only ever parses `research.md` — but read by the researcher agent to judge staleness). |
| `research.md` | New. Documents the exact `<!-- spine:research sha:... files:... -->` header `core/scripts/check-stale` parses verbatim (fixed in Phase B); this template is the canonical reference for that format, and `core/agents/researcher.md` embeds the same format inline so the agent doesn't need a second file read to produce it correctly. |
| `plan.md` | New. Documents the exact `## Predicted touch` section `core/scripts/conformance` parses verbatim (fixed in Phase B); 200-line cap and latitude table stated in-file. |
| `deviations.md` | New. Fixed the mechanical `- Status: open` line `/ship`'s merge gate greps for — see §3, this went through one real bug-fix during this phase. |
| `verify.md` | New. Structure `core/skills/verify/SKILL.md` fills in from `floor`'s `--out` JSON, `conformance`'s `--out` JSON, and each adversary's filtered verdict file. |
| `briefing.md` | New. The ≤1-page delta briefing structure. |
| `decision.md` | New. `docs/decisions/<date>-<slug>.md` structure, distilled by `/ship`. |
| `CLAUDE.md` | New. The always-loaded template, 43 lines as committed (cap is 60) — see §2 for the audit and §3 for a real path-reference bug caught and fixed while writing it. |

**`spine/core/skills/`** — all nine, `disable-model-invocation: true` on every one except `security-checklist` (`user-invocable: false`, background knowledge only):

| Skill | Status |
|---|---|
| `bootstrap` | New. Greenfield installer — charter first (before any code exists), then Layer 1/2/3 calibration, adapter generation + conformance, install wiring, hands off to `/task`. |
| `adopt` | New. Existing-repo installer — inspect-then-confirm calibration, adapter generation + conformance, the bounded survey (map + charter draft), same install wiring. |
| `task` | New. The spine's state machine — classify → research → plan (human approves) → implement (latitude table, circuit breaker) → verify → ship. Drives `.spine/current-task`, `work/<id>/state`, `work/<id>/class` — the three files the Phase B hooks read. |
| `verify` | New. Dispatches `floor`, runs the adversary agents per Layer 1 ceremony calibration, runs `verdict-filter` and `conformance`, assembles `verify.md`. |
| `ship` | New. The merge gate (floor PASS + zero open deviations, or `--bypass` recorded loudly), decision distillation, the delta briefing, the `Spine-Task:`/`Spine-Bypass:` commit trailer, ledger finalize. |
| `ratchet` | New. Converts a twice-recurring finding into an adapter check, a test, a `protected-paths.conf` entry, or (last resort, deletion-paying) a project-owned rule/CLAUDE.md line. |
| `remap` | New. `context: fork` — regenerates `docs/map.md` in isolation from the calling conversation. |
| `costs` | New. Untracked-commit ratio surfaced first (build prompt §2.6), then `ledger aggregate`. |
| `security-checklist` | New. Authorization / data-integrity / injection checklist, preloaded into the security agent — see §2, empirically confirmed preloadable despite `user-invocable: false`. |

**`spine/core/agents/`** — all three, `model: inherit` present (the v1/v2 cross-model-routing insertion point the build prompt names):

| Agent | Status |
|---|---|
| `researcher.md` | New. `tools: Read, Grep, Glob, Bash`; `disallowedTools: Edit, Write, NotebookEdit`. Strictly read-only — see §3 for how it still produces `research.md` despite that. |
| `falsifier.md` | New. `tools: Read, Grep, Glob, Bash, Edit, Write`; `isolation: worktree`. The one agent with real mutation tools, reconciled with "read-only" via worktree isolation — see §2 and §3. |
| `security.md` | New. Same read-only shape as `researcher.md`, plus `skills: [security-checklist]`. |

**`spine/README.md`** — new. Install → first task in one sitting, layout overview, update story (`git pull` in the checkout, zero project commits).

**`spine/.claude/skills/*` and `spine/.claude/agents/*`** — new, **not in the literal deliverable manifest**, added to resolve a real bootstrapping gap — see §3.1. Nine skill symlinks, three agent symlinks, all relative (`../../core/skills/<name>`, `../../core/agents/<name>.md`), so the whole checkout stays relocatable as a unit.

Nothing else was written to either repository. `spine/docs/tradeoffs.md`, the stack-independence audit, the two-stack validation, and the real `/adopt` run are Phase D.

## 2. What was verified firing

**Primitives Phase C newly depends on, verified against live docs (code.claude.com, fetched fresh — not training-data recall) before designing around them, per §0.5:** `context: fork` semantics (runs as a background subagent by default, no access to conversation history, `background: false` available if needed), `disable-model-invocation`/`user-invocable` exact truth table (confirmed `user-invocable: false` skills stay preloadable — `disable-model-invocation: true` is what blocks preloading, not `user-invocable: false`), the `skills:` agent-frontmatter field (full content injected at startup), and the background-subagent tool set (confirmed `Write`/`Edit` are in it — load-bearing for `/remap` writing `docs/map.md` while backgrounded).

**Four things that were load-bearing enough to fire-test empirically, not just doc-read, mirroring Phase A/B's own discipline:**

1. **Skill preload into a custom agent.** Throwaway `user-invocable: false` skill + throwaway agent with `skills: [that skill]`, fresh `claude -p` subprocess, Agent tool call. The agent correctly reported the skill's embedded marker string it was never told directly — **confirmed working exactly as documented.**
2. **`context: fork` isolation + background write capability.** Two-turn scripted session (`claude -p` then `claude -p --continue`): told a secret in turn 1, invoked a throwaway `context: fork` skill in turn 2 instructed to report whether it knew the secret and to write a proof file. It correctly reported not knowing the secret **and** the file was genuinely written — confirming both the isolation-from-conversation-history property and that `Write` really is available to a backgrounded fork, non-interactive `-p` mode returning the result synchronously as the docs describe.
3. **The real `researcher` agent, invoked for real** against `spine/`'s own tree (`Task: understand what core/scripts/q does`). Produced a fully correct `<!-- spine:research sha:... files:... -->` header (including gracefully handling the discovery that `spine/` itself has zero commits — see §3.3), properly cited file:line evidence throughout, stayed read-only, and surfaced a real, useful finding (`q` is currently unreferenced by any skill — see §3.4). This is strong evidence the agent file's instructions are followable, not just syntactically valid.
4. **`isolation: worktree` on the real `falsifier` agent**, in a proper scratch repo with a commit (spine's own lack of commits blocked the first attempt against `spine/` itself — see §3.3, a new primitive fact this surfaced). Confirmed via `git worktree list`: the agent ran in a genuine separate worktree, its `Write`-tool file never leaked into the original checkout, and it replied with exactly the requested JSON verdict shape.
5. **The real `security` agent's `skills:` preload**, separately, in its own scratch repo: asked it to quote back the first heading from its preloaded `security-checklist` content — it correctly returned `## Authorization` (the checklist's real first heading) and replied in the exact JSON shape.

All scratch fixtures (`/private/.../scratchpad/phaseC-primitive-test`, `.../falsifier-smoketest`, `.../security-smoketest`, and `spine/work/99990101-*` throwaway task folders) were fully deleted after each test — confirmed via the commands' own output, none persist on disk.

**New primitive fact, not previously known (worth Phase D and any future session remembering):** `isolation: worktree` requires the repository to already have at least one commit — it fails to launch with `Failed to resolve base branch "HEAD": git rev-parse failed` on a repo with none. This matters beyond spine's own accidental zero-commit state (§3.3): a truly greenfield `/bootstrap` target has zero commits until `/bootstrap`'s own step 6 setup commit lands, so **no Class 1/2 task should reach `/verify` (and therefore the falsifier agent) before that first commit exists** — `bootstrap/SKILL.md` already sequences the setup commit before handing off to `/task`, so this should already be covered, but it has not been exercised end-to-end against a genuinely commit-less project; flagged for Phase D to watch for, not assumed safe by construction.

**Self-hosting symlinks verified functionally**, not just created: `claude -p "/costs"` from a fresh subprocess in `spine/` (`--permission-mode bypassPermissions`, since `/costs` legitimately needs unprompted `Bash`) correctly resolved `${CLAUDE_SKILL_DIR}/../../scripts/ledger` through the `.claude/skills/costs -> ../../core/skills/costs` symlink and ran both `scan-untracked-ratio` and `aggregate` for real, reporting the ratio first per the skill's own instructions. Same resolution mechanism is what every other skill's `${CLAUDE_SKILL_DIR}/../../{scripts,templates}/...` reference depends on — confirmed once here rather than per-skill, since it's the same substitution every time (Phase A already confirmed the general `${CLAUDE_SKILL_DIR}` symlink-resolution mechanism; this is the first time it was exercised through the *self-hosting* symlink layer specifically, which didn't exist before this phase).

**Cross-reference audit, mechanical, not eyeballed:** every `${CLAUDE_SKILL_DIR}/../../templates/<name>` and `.../scripts/<name>` path referenced anywhere in `core/skills/*/SKILL.md` was grepped and diffed against the real contents of `core/templates/` and `core/scripts/` — zero dangling references.

## 3. Decisions made and reasoning

### 3.1 The bootstrapping paradox and the self-hosting symlinks

`/bootstrap` and `/adopt` install the `.claude/{skills,agents,rules,hooks}` symlinks *into a target project* — but that means neither skill can be invoked from inside the target project before it's installed, and neither existed as an invocable skill anywhere until this phase. Resolution (not specified by the build prompt, a necessary builder decision): both take an explicit `--project <path>` argument (matching the convention every Phase B script already uses) and are meant to be invoked from **a session running inside the `spine/` checkout itself**. That only works if `spine/`'s own `.claude/skills/` and `.claude/agents/` are wired the same way a project's are — so this phase added exactly that, self-referentially (`spine/.claude/skills/<name> -> ../../core/skills/<name>`). This is why `README.md` frames "clone once, run `claude` from inside it" as step 1. Recorded here because it's a real, load-bearing addition beyond the literal deliverable manifest, and Phase D's stack-independence/portability audits should treat it as in-scope.

### 3.2 Agents are read-only investigators; the orchestrating skill does the writing

The build prompt calls all three agents "read-only tools" but also expects `research.md` and the adversary verdict files to exist as real artifacts. Resolution: `researcher`/`falsifier`/`security` never call `Write` on the artifact they're producing — their entire final reply *is* the artifact's complete content (research.md's full text with header, or the one JSON verdict object), and the calling skill (`task` for research, `verify` for the two adversaries — both running in the main session, which does have `Write`) persists it verbatim. This satisfies "read-only tools" literally, keeps the fresh-context subagents from needing any tool-permission exception, and was the design the empirical researcher-agent test (§2, item 3) validated directly — its reply was written to disk exactly as returned, no paraphrasing needed.

**`falsifier` is the one deliberate exception**, because mandate (b) — stub the feature logic out, show the tests still pass — genuinely requires editing code and running it, not just reading. Reconciled with `isolation: worktree`: it gets real `Edit`/`Write`/`Bash` tools, but scoped to a disposable worktree copy that's never the tree the human or `/ship` will actually commit. "Read-only" here means "cannot affect the real task's working tree," not "cannot execute any command that writes bytes anywhere" — a distinction the build prompt doesn't spell out but that reading §2.5's falsifier mandate against the "read-only tools" line in §3's deliverable manifest requires resolving one way or the other. §2 item 4 confirms this actually holds (isolation genuinely prevented leakage).

### 3.3 A real, disclosed fact about this repository: `spine/` itself has zero commits

Discovered via the empirical `/costs` test (§2): `git log` on `spine/` fails with "does not have any commits yet." Everything Phase A and Phase B built exists only as untracked/uncommitted working-tree state. This is not something this phase fixed — **no commit was made**, per this session's standing instruction to only commit when the user asks, and creating spine's first commit is exactly the kind of decision (what goes in it, when) that belongs to the user or to a later phase's explicit "commit the install" step, not to an incidental discovery mid-Phase-C. It's recorded here because it's the direct cause of the `isolation: worktree` failure in §2 (worked around with a proper scratch repo for that test) and because Phase D should decide deliberately when `spine/`'s own first commit happens, rather than have it happen as a side effect of something else.

### 3.4 `q` is real but currently unreferenced

The empirical researcher test (§2) confirmed, by actually searching the repository, that `core/scripts/q` is not invoked by any script, skill, agent, or hook built so far. This is by design, not a gap: `q`'s own header comment frames it as a convenience wrapper "for ad-hoc use by skills or an engineer," and every skill built in this phase calls `core/scripts/*` capability/utility scripts directly — each of which (per `ADAPTER-CONTRACT.md` and their own Phase B implementations) already produces one-line-success/full-diagnostics-on-failure output natively, so wrapping them in `q` would add nothing. Recorded so a future reader doesn't mistake "unreferenced" for "forgotten."

### 3.5 Two internal-consistency bugs found and fixed while writing (not pre-existing — introduced and caught within this same phase)

1. **`deviations.md` template's `Tier`/`Status` fields originally used inline-code backticks around the placeholder value list** (e.g. `` - Status: `open | resolved` ``). A model filling this in literally could plausibly have kept the backticks around its chosen word, which would silently break `/ship`'s merge gate — a literal `grep '^- Status: open'`. Fixed: the template now says plainly to pick one bare word and delete the others, with an explicit comment stating the exact literal line format the gate depends on. No script or hook currently exercises this path yet (no real task has run), so this was caught by re-reading the template against the gate's actual grep command, not by a failing test — worth Phase D's worked example specifically checking this renders correctly in a real run.
2. **`CLAUDE.md`'s template originally told the installed project to reference `core/scripts/floor` and `core/rules/migrations.md`** — paths that exist inside the `spine/` checkout but **not** inside an installed project, where only `.claude/{skills,agents,rules,hooks}` are symlinked (Phase A §4) and `core/scripts/` is reached only *from inside a skill body* via `${CLAUDE_SKILL_DIR}`-relative resolution, deliberately with no second symlink (Phase A's explicit reasoning). Fixed: `CLAUDE.md` now describes the floor as something `/task` runs automatically rather than giving a raw path, and points at `.claude/rules/migrations.md` (the real, correctly-resolving symlink) instead of `core/rules/migrations.md`. Caught by re-deriving what paths actually exist in an installed project from Phase A's install-mechanism section, not by running `/bootstrap` for real (that's Phase D) — so this is a static-reasoning catch, not an empirically-exercised one; worth Phase D's real install confirming the rendered `CLAUDE.md` is actually correct on disk.

### 3.6 `/ratchet` may not write new rules into the shared `spine/core/rules/`

Caught during review, same category as §3.5: a first draft of `ratchet/SKILL.md`'s tier 4 said "a `core/rules/*.md` path-scoped rule" without qualification. Since `spine/core/` is the *shared* checkout every installed project's hooks/skills/rules symlink back to, a `/ratchet` run inside one project writing there would silently change behavior for every other project on the same machine referencing this checkout — never what a routine project-level ratchet should do. Fixed: tier 4 now specifies a **project-owned** `.claude/rules/*.md` file (a real file, not a symlink) for anything project-specific, and notes that a genuinely universal, stack-blind addition to the shared core is a spine-maintainer action, not something `/ratchet` does automatically.

### 3.7 Ledger phase-key convention (not specified by the build prompt at this level of detail)

`core/scripts/ledger harvest` (Phase B) writes `.phases[$phase].tokens`, overwriting rather than accumulating — so multiple token sources attributed to the same phase key would silently discard data rather than sum. Decided: each phase transition's `ledger mark` call harvests the phase it's closing (its own stored mark timestamp to the timestamp just written) under that phase's own key — `classify`, `research`, `plan`, `implement`, `verify`, `ship` — and each subagent invocation (`researcher`, `falsifier`, `security`) is harvested separately, in full, under its own distinct key (`research-agent`, `falsifier`, `security`) from its own subagent transcript file rather than folded into the orchestrating phase's window. `ledger aggregate`'s summation is key-name-agnostic (`.phases | to_entries | map(...)`), so this doesn't break rollup — it just means `/costs`' per-task token total is the sum of more, finer-grained line items than a naive one-key-per-phase scheme would produce. Stated explicitly in both `task/SKILL.md` §6 and cross-referenced from `verify/SKILL.md` and `ship/SKILL.md` so the convention has one source of truth.

### 3.8 `/ship`'s merge gate stays deterministic-only, deliberately

Re-reading build prompt §2.7 ("If your build adds a fourth recurring demand on the engineer's attention, you have made an architecture change — stop and raise it") against §2.5's adversary layer: nothing in the build prompt actually gates merge on adversary severity — only the deterministic floor and "open deviations block" are named as merge gates. Decided **not** to add a human confirmation step for unresolved high-severity adversary findings before `/ship`, since that would be a fourth recurring touchpoint beyond declare-class/approve-plan/read-briefing. Instead, `verify/SKILL.md` and `ship/SKILL.md` both frame high-severity verdicts as something the *model* is expected to act on mid-flow (loop back to implement, or write a deviations.md resolution) before ever calling `/ship` — a model-judgment expectation stated in prose, not a mechanical gate. This is a genuine, disclosed judgment call, not a build-prompt-mandated design; it should be named explicitly in `docs/tradeoffs.md`'s plan-adequacy-gap section in Phase D, since it's structurally the same kind of gap ("nothing mechanically stops a human from shipping something an adversary flagged") the build prompt already concedes for plan review.

### 3.9 Concrete initial values for two open questions the skills needed *now* to be runnable

Build prompt §5 assigns these to `docs/tradeoffs.md` (Phase D) for formal defense, but `/task`'s classify and research steps need concrete behavior today, not a deferred answer:

- **Class 0 threshold (§5.2):** suggested (human still confirms) at ≤2 files, ≈15 lines, no new public symbol, no protected-path touch.
- **Research-lite vs. full research (§5.5):** Class 1 grounds only directly-touched/directly-called files; Class 2 surveys the real subsystem and its actual callers, no limit.

Both are stated in `task/SKILL.md` as the working rule, explicitly flagged in-line as an initial value for `/costs` data and `docs/tradeoffs.md` to revise, not a final answer.

## 4. What Phase D needs to know that isn't obvious from the files

- **`spine/` has zero commits (§3.3).** Decide deliberately when its first commit happens; don't let it happen as a side effect of something else. This also means nothing in this checkout has been exercised through a real `git diff`/`git log`-dependent path (e.g. `conformance`, `ledger scan-untracked-ratio`) against its own history yet — only against `horizon` (Phase B) and disposable scratch repos (this phase).
- **Possible skill-name collision:** live Claude Code docs (fetched fresh this phase) mention "the bundled `/verify` and `/code-review` skills" as an example of `disable-model-invocation: true` skills that can't be preloaded — implying some Claude Code configurations ship a built-in skill literally named `verify`. This session's own available-skills listing did not show one, so this is unconfirmed, not dismissed. Check for real the first time `/adopt` installs into `horizon` and a fresh session there loads skills — if a real collision exists, it needs resolving (rename, or confirm the build prompt's explicit naming wins) before Phase D can call `/verify` working.
- **Two pre-existing (Phase B) stack-independence leaks, found by this phase's early self-audit, not fixed here (not this phase's files to unilaterally rewrite mid-build):** `core/ADAPTER-CONTRACT.md` names "dart" twice in its example verdict JSON (illustrative values, borderline); `core/hooks/dep-gate` hardcodes six package-manager names (`npm|pnpm|yarn|flutter|dart pub|pod|bundle|pip`) to recognize install commands, which is real stack-specificity leaking into a core file — the letter of "no file in the spine may reference a language, framework, or tool" is violated even though the intent (recognize *any* stack's install command) is stack-general in spirit. A concrete fix to consider in Phase D's mandatory audit: move that pattern list into a project-owned config `.spine/`-generates-at-install-time file, keeping `dep-gate` itself pattern-source-agnostic. Phase C's own new files were grepped clean (§2 self-audit) — these two are Phase B's, surfaced now so Phase D's formal audit doesn't have to rediscover them from scratch.
- **`isolation: worktree` needs ≥1 commit to exist (§2)** — relevant to `/bootstrap` on a truly-empty repo; sequencing (setup commit before first `/task` reaches `/verify`) is already correct in `bootstrap/SKILL.md` but has not been exercised end-to-end against a genuinely commit-less project.
- **The two bugs in §3.5 and the ratchet fix in §3.6 were caught by re-reading, not by running** — none of the nine skills or three agents has been exercised through a real, full `/task` → `/verify` → `/ship` cycle yet. That's Phase D's worked example. Treat this phase's skills as carefully cross-checked for internal consistency, not as proven correct under real use — the worked example is where remaining mismatches (if any) will actually surface.
- **`horizon`'s `git status` should be re-checked before Phase D touches it again** — same standing caveat Phase B carried forward; nothing in this phase touched `horizon` at all (verified: this phase's only writes were inside `spine/`, plus fully-cleaned-up scratch directories under `/private/tmp/.../scratchpad/` and throwaway `spine/work/99990101-*` folders, all removed before this handoff).
