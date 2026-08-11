# BUILD PROMPT — Spine Extensions: the design stage, and multi-repo coordination

You are Claude Code. Spine exists, is installed, and has run a real task. You are extending it with two capabilities across the phased sessions in §9: **Extension A**, a design stage that makes the first session on a new project a real, reviewed design phase whose output becomes a grounding source with the same discipline as code; and **Extension B**, coordination across multiple repositories (frontend / API / database) through a workspace root and first-class contracts.

Everything you produce must be real and runnable. You are modifying a working system: **extensions that violate spine's existing constraints are worse than no extension.** A single-repo project that never uses these extensions must see zero behavioral change — verify this explicitly before delivery.

**Working directories for this build:** the `spine/` repository (receives all core changes); the existing installed project (regression check that nothing degraded); a fresh scratch directory for the greenfield worked example; and at least two real repos plus a workspace directory for the multi-repo worked example.

Read this entire document, then spine's actual `README.md` and `docs/tradeoffs.md`, before writing anything. This prompt was designed against a summary of those documents, not the documents themselves; **where they contradict this prompt on an existing-spine fact, they win — record the contradiction at Phase A.**

---

## 0. Verify your primitives — the multi-repo design stands on one verified constraint

Verify against live documentation (code.claude.com/docs) and the installed version:

1. **Settings and hooks load only from the session's startup directory plus user scope.** Directories added via `--add-dir`, `/add-dir`, or `permissions.additionalDirectories` have their `.claude/settings*.json` — and therefore their hooks — **ignored** (confirmed limitation as of Apr 2026, issue #52934). Hooks that *are* loaded fire on matched tool calls regardless of which directory the target file lives in. Re-verify both halves empirically: write a throwaway PreToolUse hook in a workspace directory, add a second directory, and confirm (a) the workspace hook blocks an edit targeting the added directory, and (b) a hook defined only in the added directory's settings does not fire. The entire Extension B enforcement model — hooks load once at the workspace root and route by path — depends on this pair of facts.
2. **`additionalDirectories` / `--add-dir` semantics**: added directories become readable and editable under the session's permission mode.
3. **CLAUDE.md from added directories is env-gated** (`CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`, semi-documented). **Do not build anything that depends on it.** Per-repo context reaches sessions through skills reading files.
4. Whether skills and agents resolve like settings (startup dir + user scope) — assumed but not separately confirmed; test it, because it determines whether the workspace needs its own skill installation or inherits spine's.
5. Hook JSON field names on the installed version; spine's install mechanism (per its tradeoffs doc) and how the workspace interacts with it.

Anything you cannot verify, you must not build on. **A system built on a hallucinated primitive is worse than no system** — and for this build, primitive #1 is the one that kills Extension B if it doesn't hold as described. If your verification contradicts it, stop at Phase A and redesign the enforcement topology before proceeding.

---

## 1. Problem framing — what breaks spine's load-bearing assumption

Spine assumes: there is a codebase, it has facts, research discovers them, plans ground in them. `research.md` carries a commit SHA and grounding files; `check-stale` quarantines research when they drift; `conformance` scores plans against diffs; `callers` computes blast radius. Every mechanism points at existing code.

**Extension A breaks the assumption temporally**: on greenfield there is no "how it works today." Spine's tradeoffs doc concedes novel design work is where it's the wrong tool. The concession was correct for v1 and is now the thing to fix — but the fix must not be a parallel system. The design principle throughout: **spine's shape survives; only the grounding source swaps.** Decisions become what files were; a decision's content hash becomes what the SHA was; the staleness cascade becomes the design-tier circuit breaker.

**Extension B breaks it spatially**: the caller of an API endpoint lives in a different checkout, so blast radius stops exactly where cross-repo breakage lives. An API change that breaks three frontend call sites currently passes verification clean. The fix routes through declared contracts, not through cross-repo symbol indexing.

**Failure modes specific to this build** (in addition to spine's original seven, which still apply):

- **Waterfall**: a design phase producing a document nobody follows, or over-architecture because more design reads as more thorough. Counter: bounded scope with a mechanical stopping rule, and a mandatory deferred-decisions index — *what was deliberately not decided, with the trigger that forces deciding it*, is the design's most valuable section, and its absence fails the stopping rule.
- **Design theater**: adversarial review that can't work because there's no code to cite. Counter: a new evidence type the verdict filter validates for shape, not an exemption from evidence.
- **Silent design rot**: a design doc disagreeing with code is the "wrong beats missing" object spine already names as its worst. Counter: an explicit decision lifecycle with a mechanical authority handover.
- **The stranger's-mess block**: multi-repo gating that lets one repo's pre-existing failures block every task touching its contracts. Counter: the edited-vs-affected rule.
- **The half-shipped limbo**: cross-repo merges are not atomic and cannot be made atomic. Counter: not pretending otherwise — staged, ordered, *visible* shipping where every intermediate state is contract-compatible by construction.

Constraints you must not break, restated from spine and binding here: three recurring human touchpoints (declare class, approve plan, read briefing — new touchpoints must be non-recurring events, and if you find yourself adding a recurring one, stop and raise it); stack independence (new capabilities go through the adapter contract and conformance suite; nothing in core names a language or tool); deterministic beats probabilistic; wrong context beats missing (every new artifact needs its invalidation story before it ships); evidence validated for shape by script, not by a model; v1 discipline with a named v2 shelf; **do not degrade brownfield single-repo**.

---

## 2. Architecture (firm; improvable with recorded justification)

### Extension A — the design stage

**One new skill. Zero new artifact types. Zero new agents.** The design stage authors decision records *before* code, into the same `docs/decisions/` store that spine already distills them into *after* deviations. One store, two entry paths. Do not create a separate design-document format — two sources of architectural truth with independent rot rates is the failure this extension exists to prevent.

**`/design`** (disable-model-invocation; interactive — the human is present, this is a facilitated conversation, not a batch generation). Runs after `/bootstrap` calibration on greenfield; may also run on brownfield to make implicit architecture explicit, though that is not this build's worked example. Output, as files:

- `docs/charter.md` — unchanged mechanism.
- `docs/decisions/D-<seq>-<slug>.md` — foundational decisions: state management idiom, persistence approach, module boundaries, error-handling convention, auth model, repo topology. Required fields: **decision; alternatives rejected, with reasons; scope** (paths/modules governed — greppable globs, which class escalation and future rules can consume); **status**; and, when relevant, the contracts it implies.
- `docs/decisions/DEFERRED.md` — every deliberately-undecided item with the **trigger** that forces deciding it ("first task needing offline support decides D-9"). An empty deferred list fails the stopping rule.
- `work/<milestone-id>/milestone.md` for **milestone 0: the walking skeleton** — always the thinnest end-to-end path through the decided topology.
- `.spine/capabilities.json` with planned statuses: the stack is itself a design decision, so **adapters for code-independent capabilities (typecheck, lint, secret-scan, dep-diff) are generated and conformance-tested at design time**; code-dependent capabilities are marked `unavailable (no code yet — skeleton target)`.

**Decision lifecycle** — the authority handover, mechanical:

`proposed → adopted → implemented → superseded`

- **Adopted**: governs planning; a grounding source; nothing can falsify it (there is no code to disagree).
- **Implemented**: when `/ship` merges a task whose plan cited the decision, `/ship` records the implementing paths onto the record. **From that commit forward, code is authoritative.** `check-stale` treats the decision like research: implementing-path drift flags it for reconcile — amend, supersede, or reaffirm, the charter's existing protocol. A flagged, unreconciled decision appears loudly in the next briefing and blocks any *new* research from grounding on it; it never silently feeds a plan.
- **Superseded**: append-only. A new record supersedes by reference; the old record's only permitted edit is its status line. Editing decisions in place is forbidden because it destroys the semantic meaning of grounding hashes — a hash change must mean the decision *changed*, not that a typo was fixed. (Typo-level fixes: supersede with a note, or live with the typo.)

**Grounding generalization**: `research.md`'s header gains `grounding-decisions:` — decision IDs with content hashes — alongside the existing commit SHA and file list. `check-stale` gains one branch: cited decision's hash mismatched *or* status superseded → quarantine the research, exactly as file drift does. The cascade (superseded decision → quarantined research → invalidated plans citing it) is deliberate: it is spine's circuit breaker operating at the design tier, and it is the answer to "the design turns out wrong three weeks in."

**Design review** — the existing falsifier and security adversary gain design-mode mandate sections in their agent files; they run against the decision set at the end of `/design`, in fresh contexts, receiving the charter and decision records only:

- Falsifier (design mode): *construct a concrete scenario — a user action, a data shape, a failure event — that the decision set cannot handle or handles two contradictory ways, citing the decision IDs walked through.*
- Security adversary (design mode): attack the auth model, data-integrity guarantees, and trust boundaries **as decided** — design time is when security review is cheapest and highest-leverage.
- **Verdict filter extension**: one new evidence type, `decision:<id>` with a quoted span. The filter validates mechanically that the ID resolves to an existing record and the quote appears verbatim in it. Shape is validated by script; substance is triaged downstream — the identical split code verdicts already have. Verdicts with unresolvable IDs or non-matching quotes are dropped before any model reads them.

**Stopping rule** — all four checks scriptable; `/design` refuses handoff until they pass: (1) milestone 0 defined with capability targets; (2) every capability in the manifest has a planned status; (3) every foundational category either decided or in DEFERRED.md with a trigger; (4) adopted-decision count ≤ a cap (default 12 — the plan-cap logic: exceeding it means deciding things code should decide). Publish the cap as a calibration default the engineer can adjust.

**Walking skeleton definition-of-done**: shipping milestone 0 requires `test`, `smoke-seed`, `smoke-run`, `smoke-golden` conformance-passing and flipped to `implemented` in the manifest. The skeleton is done when the floor is real. This is deliberate pressure on spine's least-validated area — the smoke lane was `unavailable` on the reference install; this build must exercise it for real in the greenfield worked example, and if it fails, that failure is a finding for the tradeoffs doc, not something to route around.

**Milestone container**: `work/<milestone-id>/milestone.md` — ordered member task IDs, inter-task contracts (what task N may assume task N−1 left true), done-definition. `/task` accepts a milestone argument and loads it into planning context. `/ship` on the final member task checks the done-definition. Member tasks keep individual plans and approvals; milestone review is a non-recurring event in the charter-authorship category.

### Extension B — multi-repo coordination

**The workspace root — the topology primitive #1 dictates.** A coordination directory is the session's startup directory. Member repos join via `permissions.additionalDirectories` in the workspace's own settings. All spine hooks load once, from the workspace, and **route by path**: `workspace.json` maps each repo to path, role, and protected-path config; the three existing hooks consult it and apply the owning repo's policy to any edit by target path. No hooks are installed per-repo for workspace use, because per-repo hooks verifiably do not fire in this topology. Projects without a workspace are untouched.

**`/workspace`** (new skill): initializes the root — `workspace.json`, settings with member directories, workspace-level `work/` and `contracts/`, a thin workspace CLAUDE.md (pointer to the system, within spine's line budget). On greenfield it consumes the design stage's topology decision mechanically — the design decides "three repos," `/workspace` builds that. On brownfield it registers existing repos; each repo keeps its own adapters and capability manifest, correctly, because the stacks differ.

**Contracts, first-class**: `ws/contracts/<name>/` holds the spec file (stack-specific content; stack-blind registry). The registry entry in `workspace.json`: producer repo, producer paths, consumer repos, spec content hash. **New capability #14: `contract-check`** — a per-repo adapter verifying that repo's conformance to a contract it produces or consumes (generated client matches spec; queries match schema; whatever the stack's truth is), under the standard adapter contract, with known-pass/known-fail conformance cases like every other capability.

**Cross-repo blast radius**: `contract-touch` (workspace-level, stack-blind script) — does this task's diff intersect any contract's spec or producer paths (hash + path intersection)? On touch: consumer repos enter blast radius; their `contract-check` runs and gates; adversaries receive the contract and consumer list, and the falsifier's cross-repo mandate is to hunt **undeclared coupling** — consumers reaching into a producer outside any declared contract. Undeclared coupling is a defect, not a blind spot to tolerate. Do not build cross-repo symbol-level callers; the registry lookup carries the value at near-zero cost, and `contract-scan` (deterministic undeclared-coupling detection) is the named v2 insertion.

**Multi-repo tasks**: one task folder at the workspace, **one plan, one human approval** — the human approves the change, not per-repo fragments. Predicted-touch lists become repo-qualified (`api:src/routes/…`). Class escalation composes as **max** across repos: a protected path anywhere escalates the task. Commit trailers carry the same task ID in every repo — spine's existing linkage primitive does the cross-repo join. Conformance runs per repo, aggregates in `verify.md`.

**Ship semantics — staged, ordered, never partial-silent**:

- The plan declares a **ship order** derived from contract direction: producer before consumer for additive changes.
- **Breaking contract changes are refused as a single task.** They decompose expand → migrate → contract across a milestone: producer ships both shapes; consumers migrate; producer drops the old. This is spine's migration discipline generalized to system scale — implement it as a rule file applying to contract specs, the same pattern as the existing migrations rule, with a deterministic floor check flagging a breaking spec change co-shipped with the switch.
- **Floor semantics with attribution**: repos the task *edits* gate on their full floor; repos merely *affected via a contract* gate on `contract-check` only. This prevents a consumer's pre-existing unrelated failures from blocking every producer task forever. Baseline failure attribution is v2; state the residual.
- The ledger records each repo's merge under the shared task ID. Between first and last merge, `work/<id>/state` reads `shipping (k of n)`. The inconsistency window is not eliminated — cross-repo merge is not atomic and you must not pretend it is — but it is **visible, bounded by the declared order, and safe by construction**, because the order guarantees every intermediate state is contract-compatible.

**Charters and maps**: one system charter at the workspace; per-repo maps as today; workspace topology is not a new document — it *is* `workspace.json` plus the contract registry. The delta briefing gains one line when a ship touches contracts: contracts touched, and any undeclared coupling the falsifier surfaced (registry coverage made visible, so neglect is loud).

### Where the extensions meet

The design stage decides topology; `/workspace` consumes it; milestone 0 on a multi-repo greenfield is one thin path through all repos — which means the skeleton exercises `contract-check` and the staged ship on day one. This is also the maximal-ceremony hazard: design plus workspace init plus skeleton before anything runs. The mitigation is sequencing, already in the design: the skeleton is the *first* built thing, the design cap bounds the delay, and ceremony that hasn't produced a running system within the first milestone has failed — say so in the tradeoffs doc.

---

## 3. Deliverable manifest

**Changes in `spine/` (core; stack-blind throughout — the stack-independence audit re-runs over everything here):**

```
core/skills/design/SKILL.md          interactive; stopping-rule enforcement; adapter generation for
                                     code-independent capabilities; invokes both adversaries in design mode
core/skills/workspace/SKILL.md       workspace init; consumes design topology on greenfield; registers
                                     existing repos on brownfield
core/skills/task/SKILL.md            extended: milestone argument; repo-qualified predicted-touch lists
core/skills/ship/SKILL.md            extended: implementing-path recording onto cited decisions; staged
                                     multi-repo ship with declared order and shipping(k of n) state;
                                     contract-coverage briefing line; milestone done-definition check
core/skills/verify/SKILL.md          extended: aggregate per-repo floors; edited-vs-affected rule
core/scripts/check-stale             extended: grounding-decisions branch (hash mismatch or superseded
                                     → quarantine); implemented-decision drift flag
core/scripts/verdict-filter          extended: decision:<id> evidence type — ID resolves, quote matches
                                     verbatim, else dropped
core/scripts/contract-touch          new: diff ∩ (contract specs ∪ producer paths) → consumer blast radius
core/scripts/design-gate             new: the four stopping-rule checks, scriptable, invoked by /design
core/hooks/phase-gate                extended: workspace.json path routing
core/hooks/path-escalate             extended: per-repo protected-path policy by target path; max escalation
core/hooks/dep-gate                  extended: same routing
core/rules/contracts.md              new rule file: expand/migrate/contract for contract specs; breaking-
                                     change-with-switch flagged at the floor
core/agents/falsifier.md             design-mode mandate section; cross-repo undeclared-coupling mandate
core/agents/security.md              design-mode mandate section
core/templates/decision.md           required fields incl. scope globs, status, supersedes/superseded-by
core/templates/DEFERRED.md           item + trigger format
core/templates/milestone.md          ordered tasks, inter-task contracts, done-definition
core/templates/workspace.json        repos (path/role/protected-paths), contract registry entries
docs/tradeoffs.md                    extended per §6
```

Capability list grows to 14 with `contract-check`; the conformance suite gains its known-pass and known-fail cases. Adapter implementations for `contract-check` are written **per repo in the worked examples**, not in core.

**Produced by `/workspace` in a workspace directory:** `workspace.json`, `.claude/settings.json` (member `additionalDirectories`, hook wiring resolving to the installed core), `work/`, `contracts/`, thin `CLAUDE.md`.

**Regression requirement:** after all core changes, the existing installed single-repo project must run a trivial task through the unmodified flow with identical behavior — no new prompts, no new artifacts, no new gates. Demonstrate this in Phase D.

**v2 shelf (forbidden in this build; list with insertion points in tradeoffs):** `contract-scan`; baseline failure attribution; decision-drift metrics in `/costs`; per-repo charters; auto-generated topology maps; brownfield `/design` worked flow; multi-workspace; anything depending on the CLAUDE.md env flag.

---

## 4. Anti-patterns — this build's own, beyond spine's originals

**The parallel truth**: a design document format existing beside the decision store. One store, two entry paths — anything else is the incoherence relocated. **The evidence exemption**: letting design verdicts skip the filter "because there's no code" — the `decision:` type exists precisely so shape validation survives. **The eager architect**: an agent that fills the design stage with decisions because more architecture reads as more thorough — the cap and the mandatory deferred index are the counters; respect them in the skill's own prompt. **The atomic pretense**: any language, state, or UI implying cross-repo merge is one event. It is k of n, visibly. **The per-repo hook**: installing enforcement where it verifiably does not fire. **The silent affected repo**: a consumer gated on nothing because "the task didn't edit it" — `contract-check` is the floor for affected repos, never zero. **The skeleton skip**: handing off from `/design` to feature work with the smoke capabilities still `unavailable` and no skeleton milestone — the design-gate script must make this impossible, not discouraged.

## 5. Open questions you must resolve and defend (in the tradeoffs doc)

1. Decision content hashing: whole-file vs. body-only (status-line edits must not change the grounding hash, or supersession itself breaks grounding — probably hash everything except the status field; decide and defend).
2. Milestone ID scheme and its relation to task IDs and commit trailers, so the ledger's untracked-commit scan still works.
3. `contract-touch` inputs: how producer paths are declared per contract, and what happens when a producer refactors paths without changing the spec (registry staleness — the registry needs its own invalidation story; design it, it is not optional).
4. Ship-order derivation: automatic from registry direction, or declared in the plan and validated against the registry? (Recommend: declared, validated — the plan is the human review surface.)
5. What `/design` does when the human disagrees with an adversary's design verdict — record-and-proceed with the verdict attached, or forced supersession? (Recommend: recorded override, visible in the briefing — same trust model as `--bypass`.)
6. Where workspace-level ledger entries live and how `/costs` aggregates across workspace and member repos.
7. The design-mode adversary invocation cost and whether Class-like tiers apply to design review itself (a 4-decision design should not pay a 12-decision review).

## 6. The tradeoffs document — extend it honestly

Add: what the design stage costs (estimate human hours interactive plus tokens, to be replaced by ledger data); the maximal-ceremony hazard where A and B meet, and the sequencing that mitigates it; the registry-neglect risk (contract coverage decays unless the briefing line is read — same engagement bet as everything else); the inconsistency window that staged ship bounds but cannot eliminate; the affected-repo residual (pre-existing consumer breakage invisible to `contract-check` passes silently until v2 attribution); the decision-store residual (a decision no task ever cites again can rot in `adopted` forever — the charter's uncollided-section problem, inherited); and **whatever the worked examples broke** — spine's most valuable finding came from its worked example failing to ship, and this build must disclose failures with the same honesty rather than smoothing them over.

## 7. Worked examples — executed, not narrated, both real

**Greenfield through the design stage**: a genuine small project (choose something with real state, persistence, and one nontrivial auth decision — a toy with none of those proves nothing), taken through `/design` → adversarial design review with at least one real filtered verdict → walking skeleton milestone → skeleton shipped with the smoke capabilities flipped to `implemented` → one post-skeleton feature task grounded on decisions, shipped. If the smoke lane fails in reality as it did on the reference install, that is a first-class finding: fix what's fixable, disclose what isn't.

**Multi-repo change across at least two real repos**: a workspace over two genuinely different stacks with one declared contract between them; a task that changes the contract additively (producer + consumer, one plan, one approval, staged ship in declared order, `contract-check` gating the consumer); and a demonstration that a *breaking* change is refused as a single task and decomposes into the expand → migrate → contract milestone. Keep the artifacts — plans, verdicts (including dropped ones, counted), ledger entries, the `shipping (k of n)` states — in `docs/example/` of the workspace.

## 8. Self-red-team before delivery

Into the tradeoffs doc: for each new gate, the laziest defeat by a tired human or a motivated agent, and whether anything detects it — including at minimum: classifying a breaking contract change as additive (what catches it?); grounding research on a decision while ignoring its quarantine flag; a `contract-check` adapter that validates too little while passing conformance; the design cap gamed by cramming multiple decisions into one record. Any hook change not watched firing in the workspace topology (must be none). Any capability marked `implemented` without conformance (none). The regression check on the existing single-repo install (§3) passed and shown.

## 9. Output protocol — stop points are session boundaries

Do not attempt this in one context. Each stop ends the session; the next begins fresh and **must resume from the repositories alone**. At every stop, write `spine/work/.build/ext-phase-<X>-handoff.md`: what was built (files, one-line status); what was verified firing (hooks triggered and observed, scripts run, with commands); decisions and reasoning, including recorded disagreements with this prompt; what the next phase needs that isn't obvious from the files.

- **Phase A**: primitive verification per §0 — especially the two-part hook topology test — plus a read of spine's actual README and tradeoffs doc with any contradictions against this prompt reconciled and reported; the build plan. **Stop for review.**
- **Phase B**: Extension A's deterministic layer — check-stale branch, verdict-filter `decision:` type, design-gate, templates, ship-side decision lifecycle — each demonstrated with a real invocation. **Stop for review.**
- **Phase C**: `/design`, milestone support, adversary design-mode mandates; then the **greenfield worked example end to end**, including the skeleton ship. **Stop for review** — this is the extension most likely to need human steering on scope.
- **Phase D**: Extension B — workspace, hook routing (watched firing across the directory boundary), contracts, `contract-touch`, `contract-check` conformance, staged ship; then the **multi-repo worked example** including the breaking-change refusal; then the single-repo regression demonstration. **Stop for review.**
- **Phase E**: tradeoffs extensions, self-red-team, v2 shelf with insertion points. **Stop. Deliver.**

Direct technical prose. Every claim about tool behavior verified or flagged. If you write something you would not defend to a skeptical principal engineer, delete it and write the honest version.
