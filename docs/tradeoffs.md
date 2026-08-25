# Tradeoffs

Honest costs and limits. See `README.md` for what this system is for — this
is where it's the wrong tool, what it costs, and what it doesn't catch.

## What this costs

**Token overhead.** Research, a plan, implementation, and two adversary
agents cost real tokens over "just fix it." Rough estimate: 2–4x for a
Class 1 task (small, self-contained, low blast radius), more for Class 2
(wider blast radius, a second approver, more likely to hit a real
deviation). `/costs` is the intended source of truth once a project has run
enough real tasks to measure its own number — treat the estimate as a
starting point, not a guarantee.

**Human minutes.** Class 1 asks for three short touchpoints — confirm the
class, read a short plan before approving it, read a one-page briefing when
it ships — a few minutes each. Class 2 adds a second approver and a higher
chance of a deviation needing a real decision, so more like 10–20 minutes.
A `/design` session (turning a charter into foundational decisions before
any code exists) is closer to a milestone-sized review than a single task —
expect 20–40 minutes reading through several category decisions and two
adversary reports, more if a decision is genuinely contested.

**`/prototype` is cheap by design.** No plan, no floor, no adversary
review — the cost is whatever it takes to build the throwaway artifact
itself, plus a few minutes writing down what it settled. If a "prototype"
session is regularly running longer than that, it has quietly become
implementation work and belongs in a real `/task` instead.

**`/wayfinder` spreads its cost across many sessions, not one.** A single
ticket resolution (the unit of work `/wayfinder` completes per session) is
usually on the order of a `/design` category — a few minutes to tens of
minutes depending on whether it's a `grilling` conversation, a delegated
`research` question, or a full `/prototype` session run in between. The
real cost worth watching is the map's *total* session count for one
effort: a map that keeps spawning tickets faster than it resolves them is
a sign the effort was never one map's worth of scope, not a reason to
push through anyway.

**Multi-repo and multi-engineer overhead scale differently.** A change
spanning repos through `/workspace` pays a second floor run per affected
repo and a contract check in both directions, but review stays flat — one
plan, one approval, regardless of repo count. Working alongside other
engineers adds a conflict check before plan approval and a couple of
re-grounding checks at ship time; those are cheap in script time, the real
cost is a colleague's attention when a plan needs a second approver.

## Where this is the wrong tool

- **Tiny repos you hold in your head.** The artifact trail (research.md,
  plan.md, decisions) exists to compensate for context loss across
  sessions and across people. A script you wrote yourself ten minutes ago
  doesn't have that problem yet.
- **Genuinely novel design work.** Research assumes there's an existing
  "how it works today" to ground in. Net-new architecture has no such
  ground truth — research would either come back empty or, worse, invent
  false grounding.

## When to abandon it

If `/costs` shows the attention cost of the three touchpoints (classify,
approve, read the briefing) regularly exceeding what a careful human review
of the same diff would have cost — after the initial calibration period,
and after `/ratchet` has had a chance to turn repeat friction into a
deterministic check — the system has failed its own test. That's a real
exit condition, not a hypothetical one.

## What makes this obsolete

The artifact trail (`docs/charter.md`, `docs/map.md`,
`research.md`/`plan.md`/`verify.md`, the briefing) exists because a model
session is a stateless function of its context window today — nothing
carries forward except what's written down. A model that reliably
maintains long-horizon state across sessions removes that constraint, and
the ceremony around producing these artifacts becomes pure cost rather
than compensation for a real limitation.

## Known limits

Conceded by design, not bugs waiting to be fixed:

- **Plan quality has no backstop.** The falsifier tests an implementation
  against the plan's own acceptance checks; nothing checks whether those
  checks were the right ones to write. A plan with weak acceptance criteria
  passes verification cleanly, every time, by construction — that load
  sits entirely on whoever approves the plan. Probably the right place for
  it (deciding whether a plan tests the right thing isn't a check that
  decomposes into a deterministic gate), but it means plan review is the
  one link in the chain with nothing mechanical behind it. An `auto`
  task compounds this: it skips plan approval entirely, so the falsifier's
  mandatory stub-out probe — the "partial backstop for the plan review
  auto skipped" — is checking a plan whose own acceptance checks were
  never reviewed by anyone. Two unbacked links stacked, not one.
- **Class 0 has no adversarial backstop at all.** A change under the
  trivial-change threshold (≈2 files, ≈15 lines, no protected path) gets
  one commit and a trace line — no research, no plan, no verify, no
  adversary. The only thing that can catch it is `path-escalate` noticing
  a protected-path touch after the fact. A small-but-wrong logic change
  (an inverted condition, an off-by-one) in an unprotected file ships on
  the model's own unreviewed judgment alone. This is the price of Class 0
  being cheap; if that price turns out too high in practice, the fix is a
  narrow one (e.g. always run the floor's lint/type layer even when
  everything else is skipped), not a redesign.
- **Class 2 is permanently unreachable for a project with no smoke-testable
  runtime.** Unlike `contract-check`/`ui-render`/`ticket-fetch`/`open-pr`/
  `worktree-prep`, `smoke-seed`/`smoke-run`/`smoke-golden` have no
  legitimate `not-applicable` escape hatch — the floor's Class 2 gate
  treats anything other than `implemented` as a hard fail (`core/scripts/
  floor`'s smoke-run check). A pure CLI/library/batch project genuinely
  has no stack to seed-run-verify against and can never pass Class 2 floor
  as a result. Deliberate (no Class-2-risk work ships without a working
  smoke harness), not an oversight, but worth knowing before adopting spine
  for a project shaped that way.
- **Adversary findings inform `/ship`, they don't gate it.** `/verify`
  logs falsifier/security findings to `verify.md`, but even a `high`
  severity finding doesn't fail verify by itself — at ship time it either
  prompts an interactive ask (`guided`) or gets auto-routed to a
  milestone's "Known gaps" list (`checkpointed`/`auto`). A human can say
  "ship it anyway, just flag it" and the merge gate doesn't stop them. The
  only hard ship-time gates are the deterministic floor, zero open
  `deviations.md` entries, and (multi-repo) contract/render checks. This
  is deliberate — a hard block on adversary opinion would recreate the
  review-bottleneck rubber-stamping this system exists to avoid — but
  it means "adversarial review" is disclosure with visibility, not
  enforcement, and should be read that way.
- **The deviation circuit breaker runs on an honor system.** It counts
  deviations actually logged to `deviations.md`; nothing forces one to get
  logged. That's a norm the skill instructions ask for, not something a
  hook enforces.
- **The deviation/setup-event split is a judgment call, not a mechanical
  test.** `core/skills/task/SKILL.md` §4 asks, per decision hit during
  implement: did this teach us the plan's understanding of *the product*
  was wrong (a real deviation — `deviations.md`, counts toward the circuit
  breaker, appears in the briefing), or only that this project's own
  tooling config (an adapter, `capabilities.json`, `protected-paths.conf`)
  was imperfect (a **setup event** — `notes.md`'s `SETUP:` line,
  `verify.md`'s "Setup events" section, never `deviations.md`, never
  counted)? Nothing stops a task from misclassifying a real deviation as
  a setup event to dodge the circuit breaker — same honor-system exposure
  as the bullet above, just one boundary test wide instead of zero. It's
  also currently prose-only visibility: `render-task`/`render-dashboard`
  surface `deviations.md`'s and `TOOLING GAP:`'s structured `ledger.json`
  fields, but nothing yet gives `SETUP:` lines the same structured field —
  a setup event is visible in `verify.md`/`briefing.md`, not (yet) in the
  dashboard or `/costs`.
- **The `implement` phase's token harvest runs on an honor system, even
  though the mark that's supposed to trigger it mostly doesn't.**
  `task/SKILL.md` §5/§6 bundles "mark `verify`" and "harvest `implement`'s
  token window into it" into one documented step; neither half is
  enforced by a hook. As of round 4, real installed-project data shows the
  two halves have diverged: the *mark* itself (`verify.marked_at`) is now
  present in ~95% of real tasks, but the *harvest*
  (`implement.tokens`) is still missing in roughly two-thirds — sessions
  reliably call `ledger mark <task-id> verify`, they just don't reliably
  also make the paired `ledger harvest` call. `render-task` surfaces
  recorded subagent cost against the umbrella phase when the mark itself
  is missing, and flags a phase's cost as "(not recorded)" when its own
  mark is present but its harvest evidently wasn't, so the spend isn't
  silently presented as zero — but the per-phase cost breakdown stays only
  as reliable as the session that ran it, and nothing currently forces the
  harvest to happen.
- **A ledger field can still go dead between audits.** `core/scripts/
  ledger-field-audit` heuristically checks that every `ledger set
  <task-id> <field>` documented in a skill has at least one reader-shaped
  reference elsewhere in the deterministic layer — added after this exact
  "written every task, read by nothing" bug recurred across two audit
  rounds (nine fields total). It's grep-based, not a real parser, and
  nothing runs it automatically — no hook, no floor, no `/task` path calls
  it. Run it by hand periodically; a field it flags is a lead to check,
  not a settled fact, and a field it doesn't flag isn't proof of a real
  reader either (a coincidental substring match reads as clean).
- **Adversary findings are checked for evidence shape, not evidence
  truth.** A finding needs a real file:line or command output to survive
  filtering — but nothing confirms the cited evidence actually supports
  the claim. A human skimming `verify.md` is the only backstop.
- **The write-blocking hooks don't see every way to write a file.** They
  recognize `Edit`/`Write` and common Bash write patterns (redirects,
  `sed -i`, `tee`, `cp`/`mv`) and fail closed when a target can't be
  confidently resolved — but a mutation shape outside that list (a custom
  wrapper, a file write buried inside another interpreter's own call) is
  invisible to them.
- **The floor trusts its own adapters between recalibrations — and
  `adapter-conformance` can't catch a fake-but-passing self-test even at
  recalibration time.** Adapter conformance is validated when an adapter
  is written or a project recalibrates — nothing re-validates it on every
  task, so an adapter hand-edited to always pass wouldn't be caught until
  the next recalibration. But `adapter-conformance` is also, by
  construction, a black-box exit-code/output-shape checker: it confirms
  `--self-test pass`/`fail` behave (right exit code, one-line output) but
  never inspects whether a self-test fixture actually proves what
  `core/ADAPTER-CONTRACT.md` §4 requires — a changed-file-set capability's
  scoping fixture genuinely including an out-of-scope violator, or a
  self-test exercising the adapter's real invocation path rather than a
  simplified stand-in. §4 cites a real incident of exactly this (a
  `callers` adapter whose self-test grepped a bare symbol name while the
  real adapter greps a full repo-relative path — a shape neither fixture
  ever exercised, so it stayed "conformant" indefinitely). A human
  reviewing a hand-written adapter is the only real backstop for those two
  rules; nothing mechanical currently checks them.
- **A per-task floor only sees the current diff.** Lint and type checks are
  scoped to changed files so a task never fails for debt it didn't write —
  the tradeoff is that pre-existing debt in untouched files stays invisible
  to the per-task floor by design. A whole-tree mode exists for periodic
  audits or CI, but nothing runs it automatically.
- **Semantic collisions between sequential tasks aren't caught.** Two tasks
  that touch logically related but textually disjoint code can each pass
  their own research/plan/verify cleanly and still combine badly.
- **A charter section that never collides with reality can rot
  undetected.** No mechanical check falsifies stated intent — the only
  invalidation channel is a deviation that happens to cite the stale line.
- **Seeded data isn't production-shaped data.** Even where a smoke-test
  lane exists for a stack, a seeded dataset only covers the shapes someone
  thought to seed.
- **Concurrency defects are out of scope.** No stress or
  concurrency-testing lane exists.
- **A briefing.md heading convention only binds tasks shipped after it
  changed.** The template's bold-label convention (`**Floor:**`,
  `**Overrides & bypasses:**`) is kept stable specifically so a future
  aggregate reader can rely on it — but that stability is prospective
  only; a real project's older briefings, written under a prior heading
  shape, don't get rewritten when the convention changes. An aggregate
  reader built later needs to tolerate the older shape too, or accept it
  will miss/misparse a project's earliest tasks.
- **Secrets, credentials, and PII handling have no auto-loaded discipline
  rule, unlike migrations/contracts/auth.** `core/rules/migrations.md`,
  `core/rules/contracts.md`, and `core/rules/auth.md` all load automatically
  when a matching path is read, because each has a reasonably reliable
  directory-naming convention to scope a `paths:` glob against. Secrets/
  credentials/PII don't — a credential can be touched from anywhere a
  request is authenticated, a row is read, or a log line is written, with
  no comparable naming convention to key off. `secret-scan` (content-based,
  every floor run) and whatever a project's own calibration added to
  `.spine/protected-paths.conf` are the two real mechanisms covering this
  today; a fourth `core/rules/` file was deliberately not written to paper
  over that gap with an unreliable glob that would look like coverage
  without actually providing it.
- **Worktree isolation (Extension D, experimental) delays registry
  visibility to a merge.** When `/task` spins up a second task into a
  worktree (`core/skills/task/SKILL.md`'s Resuming section,
  `docs/worktree-support-plan.md`), `EnterWorktree` puts that task on its
  own new branch — git can't check the same branch out in two worktrees
  at once. `registry-sync` still pushes `work/<task-id>/` to whatever
  branch is checked out, which is now that task's own branch, not the
  project's shared default. A colleague's `claims-check`/`gaps-report`
  won't see that task's registry entry until the branch merges. Harmless
  for the same engineer running two terminals on one machine (both
  worktrees share the local `.git`); a real gap for the cross-engineer
  coordination story the registry otherwise assumes.
- **`/autopilot` (experimental) removes every human stop `/task` has,
  including the ones spine treats as structural rather than stylistic —
  by explicit request, not by accident.** Class 2's forced `guided`
  autonomy and second-approver requirement, halt-tier deviations, the
  circuit breaker, `claims-check` blocks, and flag-blocked advances all
  normally exist because some decisions are judged to need a human in the
  loop, not just a slower one. `/autopilot` (`core/skills/autopilot/
  SKILL.md`, `docs/autopilot-plan.md`) self-resolves every one of them
  and defers the entire review to a single end-of-run report instead of
  per-decision, per-task review. What stays real and unweakened: the
  deterministic floor, the falsifier's stub-out probe, the security
  adversary, `claims-check`, `conformance`, and `contract-touch` — this
  removes *stops*, never *checks*, mirroring the same distinction the
  `autonomy` field already draws for ordinary `auto` tasks, just pushed
  to its extreme point. **Every commit stays local — `/autopilot` never
  pushes and never opens a PR**, unlike ordinary `auto` autonomy (which
  already pushes before its draft PR even opens); nothing this run does
  leaves the machine unreviewed, by explicit choice, not because pushing
  would have been unsafe in principle. Explicitly experimental and
  disclosed as possibly-temporary; not the default or recommended way to
  use spine.

## Working with other engineers

Real-time coordination between engineers working the same repo is built
for roughly 2–4 people, not more, and it's honest about what's actually
mechanical:

- **One layer is a real hook; everything else is a script an agent is
  instructed to act on.** The write-blocking hooks are the one tier a
  session can't simply choose to skip. A conflict check before plan
  approval, ship-time re-grounding, flag acknowledgment, and
  second-approver review are all real scripts that compute a real, correct
  answer — but acting on that answer is skill instruction, not an enforced
  gate. A session that writes the target state file directly instead of
  following the instructions can walk past any of them.
- **Git identity is a coordination primitive, not authentication.**
  `git config user.name`/`user.email` is trivially spoofable — exactly as
  trustworthy as a commit author field always was, no more.
- **What breaks past about four engineers.** Overrides and conflicts rise
  faster than one person's attention can track them, and every ship still
  produces its own briefing with nothing aggregating across them — volume
  outpaces what a human skimming for patterns can actually catch.

The mitigating fact, same as everywhere else in this system: a human reads
the plan and the briefing, and that's where a pattern of silent bypass
would actually surface.

## Cross-repo work

A change spanning more than one repository coordinates through a small
workspace root and a declared contract registry:

- **Declared surfaces, not meaning.** The conflict check only sees what a
  plan declares it will touch — a collision in undeclared territory is
  invisible to it by construction. The registry-driven check fails safe
  (treats a contract as touched) when a declared path goes missing
  entirely, but not on every possible drift shape.
- **Contract checks validate shape, not behavior.** They confirm a field
  exists with the right name on both sides of a contract, never that it's
  computed correctly. A correctly-named, incorrectly-computed field passes
  cleanly — only the adversary review, which reads for behavior rather than
  structure, catches that class of bug.
- **`contract-touch`'s breaking-change classification catches every
  removal-or-modification-shaped break, never an addition-shaped one.**
  `core/rules/contracts.md` requires a breaking spec change to decompose
  into an expand/migrate/contract milestone, mechanically enforced by
  `contract-touch` reading the spec diff — but a newly *required* field is,
  line-for-line, a pure addition, indistinguishable from a newly *optional*
  one by a diff-only heuristic. Whether a field is required or optional is
  stack-specific spec semantics this stack-blind check doesn't parse; a
  required-field addition has to be caught by plan approval or adversary
  review instead, same as any other behavior-shaped gap above.
- **A staged multi-repo ship has a real, bounded inconsistency window.**
  Between the first repo's commit and the last, an in-between state
  genuinely exists. It's safe only because the declared ship order
  guarantees every intermediate state stays contract-compatible by
  construction — the window doesn't disappear, it's just never unsafe.
- **Install ties to a machine-local path.** The core is cloned once per
  machine and referenced by an absolute path; a second engineer needs
  either the identical clone path or to re-run setup and regenerate their
  own local wiring.

## The design stage

`/design` turns a charter into a reviewed set of foundational decisions
before any code exists. Its own cost shape: contract coverage and a
project's decision store are only as good as someone keeping them current,
and nothing mechanically verifies either is complete. Composing `/design`
with a fresh multi-repo `/workspace` setup and a skeleton first milestone —
all before any code exists — is the specific ceremony-compounding risk to
watch for on a brand-new project's first day. The decision cap (a fixed
limit on how many decisions one session can adopt) and shipping a skeleton
first are the mitigations, but there's no substitute for noticing if day
one is taking longer than the work justifies.

## Deferred (not built)

Named explicitly so the boundary is clear rather than discovered by
surprise:

| Not built | Where it would attach |
|---|---|
| Parallel or team-of-agents orchestration | Every real flow — `/task`'s own phases, and `/wayfinder`'s ticket-by-ticket map — is strictly sequential: one active task, one active map, one session at a time. `/wayfinder` (below) narrows this row from where it used to stand — it adds *sequential* multi-session coordination through committed files (the same idiom `/task`'s milestone member-tasks already use, one level earlier), never parallelism. There is still no claim mechanism, no concurrent agents on one unit of work, and no cross-session locking beyond "one pointer file names the active one." |
| Parallel work streams on one task | Calibration has the field; it's hardcoded off |
| Cross-model adversary routing | Every agent inherits the session's model; nothing routes a different one in |
| Scheduled cleanup of pre-existing duplication | Duplication checks only run against a task's own changed files, never sweep existing debt |
| A product-spec layer | The charter deliberately stays at constraints, not a spec — a product spec itself stays optional and human-authored, never generated. `docs/vision.md` is the one exception, and only partly: it's still never invented outright, but `/wayfinder` (added since this row was first written) does write it, one confirmed line at a time, when the milestone shape was genuinely unknown rather than just unwritten — see `core/skills/wayfinder/SKILL.md` §4. |
| A concurrency or stress-test lane | The smoke-test capability is the insertion point if this gets built |
| Deterministic detection of undeclared cross-repo coupling | Only the adversary review looks for this today; no mechanical scan does |
| A workspace-wide cost rollup across member repos' own task histories | Each repo's own `/costs` works; nothing sums across a workspace yet |
| One repo belonging to more than one workspace | Unsupported — a workspace assumes exclusive ownership of its member repos |
| Abandoning a task (a terminal state short of `done`) | `work/<task-id>/state` today only ever reaches `done` via `/ship`; nothing lets an engineer close out a task they've decided not to finish. Deceptively not a one-file fix: at least five existing mechanisms treat "not `done`" as "still open" and would each need to learn a new `abandoned` state — `core/skills/tasks/SKILL.md`'s own open-task definition, `core/scripts/next-milestone-task`'s milestone-completeness check, `core/scripts/claims-check`'s live-claims predicate, `render-dashboard`'s timeline, and `.spine/current-task` clearing if the abandoned task is the active one. The real design fork underneath all of that: when a milestone's own member task gets abandoned, does that slot need a brand-new replacement task before the milestone can ever reach done, or does the milestone itself need re-scoping through `/roadmap`? That's a design decision on the order of choosing `/wayfinder`'s ticket types, not a mechanical add — give it its own design pass before touching any of the five files above. |
| Reverting a shipped task | No mechanism today undoes a task after `/ship` — the only path is a fresh, manually-authored task that happens to reverse the change. Not even the framing is settled yet: is "rollback" a `git revert` of the ship commit(s) (fast, but bypasses research/plan/verify for the undo itself — exactly the kind of unreviewed change spine exists to prevent), or a real compensating task that goes through the normal classify → research → plan → verify → ship discipline (safer, but slower, and still has to decide what happens to anything the original task's `plan.md` cited — a decision's `## Implementing paths`, a milestone's `## Known gaps for future member tasks` entry it resolved, a `docs/decisions/` record distilled from it)? Bigger and less scoped than abandoning a task above; needs its own dedicated design conversation, not a bolt-on. |
