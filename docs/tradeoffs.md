# Tradeoffs

Honest costs and limits. See `README.md` for what this system is for — this
is where it's the wrong tool, what it costs, and what it doesn't catch.

## What this costs

**Token overhead.** Research, a plan, implementation, and two adversary
agents cost real tokens over "just fix it." Rough estimate: 2–4x for a
Class 1 task (small, self-contained, low blast radius), more for Class 2
(wider blast radius, guided at every step, more likely to hit a real
deviation). There's no built-in instrumentation for this — measure it
yourself against real task time if the estimate matters to your team;
treat it as a starting point, not a guarantee.

**Human minutes.** Class 1 asks for three short touchpoints — confirm the
class, read a short plan before approving it, read a one-page briefing when
it ships — a few minutes each. Class 2 adds more stops (it always runs guided) and a higher
chance of a deviation needing a real decision, so more like 10–20 minutes.
A `/design` session (turning a charter into foundational decisions before
any code exists) is closer to a milestone-sized review than a single task —
expect 20–40 minutes reading through several category decisions and two
adversary reports, more if a decision is genuinely contested.

**No second approver.** Class 2 used to require a different git identity to
approve the plan. That was removed: the check compared two strings from
`git config`, which prove nothing about who reviewed anything; there was no
command for the approver and no way for them to find a pending plan; and for a
solo engineer it was a self-approval override on nearly every Class 2 task
(78 of 82 in one real set). Class 2 keeps its other protections: always
`guided`, protected-path escalation, the adversaries, and the human approving
the plan. A team that wants real cross-review should enforce it where it can
be authenticated, in branch protection on the PR.

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

If the attention cost of the three touchpoints (classify,
approve, read the briefing) regularly exceeds what a careful human review
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
- **Class 0 has no adversarial backstop, only a lint and type check.** A change
  under the trivial-change threshold (≈2 files, ≈15 lines, no protected path)
  gets one commit, a trace line and `floor 0` (types and lint on the changed
  files; the result is logged as a `floor0` event). No research, plan, verify or
  adversary. A small-but-wrong logic change that type-checks and lints (an
  off-by-one) in an unprotected file still ships on the model's own judgment;
  `path-escalate` catches a protected-path touch after the fact.
- **Class 2 is permanently unreachable for a project with no smoke-testable
  runtime.** Unlike `ui-render`/`ui-conformance`/
  `ticket-fetch`/`open-pr`/`worktree-prep`, `smoke-seed`/`smoke-run`/`smoke-golden` have no
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
  `deviations.md` entries, and the render checks. This
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
  as the bullet above, just one boundary test wide instead of zero. A
  setup event is visible in `verify.md`/`briefing.md`; there's no
  project-wide aggregate view across tasks.
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
  rules; nothing mechanical currently checks them. A narrower instance of
  the same gap — a `--self-test fail` branch whose own internal probe
  unexpectedly succeeds versus one that correctly demonstrates the
  failure, both converging on the same exit code — now has a mechanical
  backstop: the `SELF-TEST-FAIL-FIXTURE-BROKEN:` marker convention
  (`core/ADAPTER-CONTRACT.md` §4) lets `adapter-conformance` tell the two
  apart instead of reporting an identical clean pass for either. But it's
  opt-in per adapter, not a property `adapter-conformance` can verify is
  present where it should be — a fail branch with a real
  unexpected-success path that never emits the marker degrades silently
  back to the old undetectable behavior, same as before this fix.
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
  rule, unlike migrations/auth.** `core/rules/migrations.md`
  and `core/rules/auth.md` both load automatically
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
- **`/autopilot` (experimental) removes every human stop `/task` has,
  including the ones spine treats as structural rather than stylistic —
  by explicit request, not by accident.** Class 2's forced `guided`
  autonomy, halt-tier deviations, the
  circuit breaker, and the other human stops all
  normally exist because some decisions are judged to need a human in the
  loop, not just a slower one. `/autopilot` (`core/skills/autopilot/
  SKILL.md`) self-resolves every one of them
  and defers the entire review to a single end-of-run report instead of
  per-decision, per-task review. What stays real and unweakened: the
  deterministic floor, the falsifier's stub-out probe, the security
  adversary, and `conformance` — this
  removes *stops*, never *checks*, mirroring the same distinction the
  `autonomy` field already draws for ordinary `auto` tasks, just pushed
  to its extreme point. **Every commit stays local — `/autopilot` never
  pushes and never opens a PR**, unlike ordinary `auto` autonomy (which
  already pushes before its draft PR even opens); nothing this run does
  leaves the machine unreviewed, by explicit choice, not because pushing
  would have been unsafe in principle. Explicitly experimental and
  disclosed as possibly-temporary; not the default or recommended way to
  use spine.

## The design stage

`/design` turns a charter into a reviewed set of foundational decisions
before any code exists. Its own cost shape: contract coverage and a
project's decision store are only as good as someone keeping them current,
and nothing mechanically verifies either is complete. Composing `/design`
with a skeleton first milestone —
all before any code exists — is the specific ceremony-compounding risk to
watch for on a brand-new project's first day. The decision cap (a fixed
limit on how many decisions one session can adopt) and shipping a skeleton
first are the mitigations, but there's no substitute for noticing if day
one is taking longer than the work justifies.

### The M0 bootstrap waiver

`floor` fails a Class 2 task outright when `smoke-run` isn't `implemented`
(§3.8's hard gate, re-tested across four audit rounds). That is circular
for a walking skeleton: milestone 0's own job is to *build* smoke, yet its
first tasks are Class 2 by necessity — dependency manifests, `db/`, `auth/`
are protected paths — so none of them could pass a floor whose smoke layer
didn't exist yet. Found for real on the first M0 task of a greenfield
project (the workspace scaffold), which could only have shipped via
`/ship --bypass`, a rule meant for emergencies.

The fix is a narrow waiver, not a softer gate. `floor` now records
`degraded:waived-bootstrap` (a DEGRADED line, never a pass — smoke did not
run) instead of failing **only when all of these independent declarations
agree**: the run names a task whose `work/<task-id>/milestone` is exactly
`M0`; `capabilities.json` has `smoke-run` `unavailable` with a "skeleton
target" reason (the marker `/design` §2 writes — not `missing`, not
`not-applicable`, the wrong shapes the gate was hardened against); and
`work/M0/milestone.md`'s Capability targets row for `smoke-run` reads
`unavailable*`. Remove any one and the hard gate applies unchanged;
`core-selftest` has a case per missing fact, and reverting the waiver to
"always grant" fails five cases.

What this costs, disclosed rather than discovered later:

- **M0 tasks before smoke lands ship without it** — that is the point, and
  every one carries the DEGRADED line into its briefing's Floor bullet.
- **The waiver does not know which M0 task is last.** A final member task
  that still hasn't built smoke passes the floor; the backstop is `/ship`
  §3b's done-definition check, which reports "not met" loudly rather than
  treating milestone completion as automatic. That check reports, it does
  not block.
- **It trusts two files the project itself writes** (`capabilities.json`,
  `work/M0/milestone.md`). Both are committed and reviewed at `/design`
  time, and `design-gate` already forces the milestone table to exist, but
  a hand-edit that fabricates both agreeing declarations would defeat it.
  Same residual as every other declared-surface mechanism here.
- **M1+ is never waived.** A later milestone that copies M0's table gets
  the hard gate.

## Visual fidelity and content provenance (`ui-capture`, `ui-fidelity`, `content-sources-check`)

Design rationale: `docs/proposal-ui-fidelity.md`. Two real misses drove it: a
screen that passed `ui-conformance` while looking nothing like its mockup
(wrong layout, wrong button variant, a component rendering its own props run
together, an unused badge prop), and invented copy/numbers in fixtures that
no step could see.

- **A scripted capture plus a reviewer, not a scripted comparison.** §4's
  self-test needs a guaranteed-pass and guaranteed-fail fixture; "does this
  match the mockup" has neither without pixel-diffing, which §3.3/§3.9
  reject. `ui-capture` is deterministic and self-testable (it renders each
  state and records measurable facts); the judgment is a fresh-context
  `ui-fidelity` agent per screen.
- **Findings must cite a captured fact (`render` evidence).** `verdict-filter`
  checks the quoted span exists in the capture, the same bar `decision`
  evidence gets. Recall is deliberately capped: a visual difference no
  captured measurement expresses is dropped. This trades misses for trust;
  it is unmeasured until the calibration run in the proposal's acceptance
  section is done, and should be treated as unproven until then.
- **Not a floor gate, but never unanswered.** Findings don't change
  `/verify` PASS/FAIL (a model's judgment shouldn't fail the floor); a
  failed capture does. `/ship` §3a requires a human disposition (fix now,
  carry, decline-with-reason) for every kept unfixed finding at every
  autonomy level — a deliberate exception to `checkpointed`/`auto`'s
  no-scheduled-stop rule, since this is the only check that notices a
  wrong-looking screen.
- **Gallery routes or step files, adapter's choice.** A project that builds a
  dev-only state gallery (`/__ui/<screen>?state=<state>`, from typed test
  data) uses it and skips step files; `default` is still captured from the
  real signed-in route. A gallery proves the view can look right in a state,
  not that the app reaches it — disclosed in the contract. The capture
  adapter consumes the project's signed-in context and gallery rather than
  building its own.
- **Every declared state is compared, or reported not compared with a
  reason** (`no_screenshot`, `no_driver`, `driver_failed`). Reaching a state
  needs hand-authored `docs/ui/states/<id>.json` steps, which go stale when
  labels change (surfaced as `driver_failed`, never a pass). Chosen over a
  `?state=` hook because it needs no product code.
- **Content provenance is a plan-time gate.** `## Content sources`
  (MACHINE fence, `content-sources-check`) requires every block of copy or
  numbers to cite a real file, a human-supplied value, or `none` (a hard
  stop-and-ask, at every class and autonomy). Cited sources must be in
  `research.md`'s `files:`, which is what puts the spec and screenshots
  under `check-stale`. The fidelity reviewer also flags render text found in
  neither the spec nor the screenshot — screenshots aren't machine-readable,
  so that part is the vision reviewer's, not a script's.
- **`.spine/ui-content-paths.conf`** makes content-only diffs (fixtures,
  seed data, copy) count as UI touches in `ui-touch`, which previously fired
  only on view-file globs.
- **Rejected:** a falsifier mandate for untraceable strings (duplicates the
  plan gate plus the reviewer); a write hook keyed on `*.fixtures.*` (a
  filename convention is per-project and core is stack-blind); a CLAUDE.md
  "never invent content" rule as the only defense (advisory).
- **Not yet verified:** no project has a `ui-capture` adapter yet, so the
  agent and step 1e have never run end to end. The core scripts are covered
  by `core-selftest`; the agent's quality is not.

## Deferred (not built)

Named explicitly so the boundary is clear rather than discovered by
surprise:

| Not built | Where it would attach |
|---|---|
| Parallel or team-of-agents orchestration | Every real flow — `/task`'s own phases, and `/wayfinder`'s ticket-by-ticket map — is strictly sequential: one active task, one active map, one session at a time. `/wayfinder` (below) narrows this row from where it used to stand — it adds *sequential* multi-session coordination through committed files (the same idiom `/task`'s milestone member-tasks already use, one level earlier), never parallelism. There is still no claim mechanism, no concurrent agents on one unit of work, and no cross-session locking beyond "one pointer file names the active one." |
| Cross-model adversary routing | Every agent inherits the session's model; nothing routes a different one in |
| Scheduled cleanup of pre-existing duplication | Duplication checks only run against a task's own changed files, never sweep existing debt |
| A product-spec layer | The charter deliberately stays at constraints, not a spec — a product spec itself stays optional and human-authored, never generated. `docs/vision.md` is the one exception, and only partly: it's still never invented outright, but `/wayfinder` (added since this row was first written) does write it, one confirmed line at a time, when the milestone shape was genuinely unknown rather than just unwritten — see `core/skills/wayfinder/SKILL.md` §4. |
| A concurrency or stress-test lane | The smoke-test capability is the insertion point if this gets built |
| Abandoning a task (a terminal state short of `done`) | `work/<task-id>/state` today only ever reaches `done` via `/ship`; nothing lets an engineer close out a task they've decided not to finish. Deceptively not a one-file fix: at least two or three existing mechanisms treat "not `done`" as "still open" and would each need to learn a new `abandoned` state — `core/scripts/next-milestone-task`'s milestone-completeness check, `phase-gate`'s view of which tasks may write, and `.spine/current-task` clearing if the abandoned task is the active one. The real design fork underneath all of that: when a milestone's own member task gets abandoned, does that slot need a brand-new replacement task before the milestone can ever reach done, or does the milestone itself need re-scoping through `/roadmap`? That's a design decision on the order of choosing `/wayfinder`'s ticket types, not a mechanical add — give it its own design pass before touching any of the three mechanisms above. |
| Reverting a shipped task | No mechanism today undoes a task after `/ship` — the only path is a fresh, manually-authored task that happens to reverse the change. Not even the framing is settled yet: is "rollback" a `git revert` of the ship commit(s) (fast, but bypasses research/plan/verify for the undo itself — exactly the kind of unreviewed change spine exists to prevent), or a real compensating task that goes through the normal classify → research → plan → verify → ship discipline (safer, but slower, and still has to decide what happens to anything the original task's `plan.md` cited — a decision's `## Implementing paths`, a milestone's `## Known gaps for future member tasks` entry it resolved, a `docs/decisions/` record distilled from it)? Bigger and less scoped than abandoning a task above; needs its own dedicated design conversation, not a bolt-on. |

## Human touchpoints (`human-touchpoint.md`, `touchpoint-lint`)

Design rationale: questions and end-of-phase reports were reaching the
human full of spine's own vocabulary (`check-stale`, `D-24`, `gap-10`,
`Class 2`) and option labels that described mechanism, not consequence.
Most of the worst ones were not written anywhere in spine: at a stop where
a skill only said "tell the human what you need resolved," the model
improvised the question, jargon included. The fix therefore puts fixed
wording (a marked block: decision, why it's yours, what to know, a
recommendation, options by consequence) at every stop site, and a
matching `report` block for the messages that end a phase.

What is checked mechanically, and what is not:

- **Checked** (`core/scripts/touchpoint-lint`, run by `core-selftest`):
  every marked block in `core/skills/*/SKILL.md` has its required labels,
  every option states `next:` and `undo:`, and every glossary term and
  `D-<n>` / `gap-<n>` / `M<n>` id is glossed inline at first use. The
  messages hooks and scripts show a human are run on fixtures and linted
  the same way; a skill that cites the standard but contains no block
  fails.
- **Not checked**, deliberately: whether an *unmarked* stop exists (the
  words "stop", "ask" and "wait" mean too many things in these skills for
  a grep to be reliable, and a noisy lint gets ignored), whether the
  wording is actually clear, and what the model says at run time. The
  block gives the model fixed text to fill in instead of composing its
  own, which is the mitigation; a stop added without a block is caught in
  review, or by `/ratchet` if it recurs.
- **Placeholders are unchecked:** the lint reads the fixed wording, not what
  gets filled into `<...>` at run time, so "say the milestone's title, not
  `M2`" is a rule for the writer.
- **Plain-word limit:** the glossary is a closed list, so a new internal
  term is invisible to the lint until someone adds it.

Quick yes/no prompts (cheap to undo, confident recommendation) use a compact
`confirm` block — a question, two answers, one marked recommended, six lines
at most — after the full block proved too heavy for them; anything with
real consequences keeps the full block.

Two behavior changes rode along, both approved: `/task` no longer asks the
human whether to redo research when `check-stale` flags only the task's own
placeholder-to-task-id edit in `milestone.md` (it keeps the research and
notes the false positive); and `core-selftest` now exits non-zero when a
case fails (it previously ended with whatever its last command returned).
