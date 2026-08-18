# Proposal: `/intake`, adaptive autonomy, and `/spine` — right-sizing human attention

Status: PROPOSED — 2026-08-18. Direction confirmed with the user; entrypoint
fork resolved (unified `/intake`-into-pipeline); gap handling resolved (§7).
Revised 2026-08-18 after the user noted the org already enforces ticket-in-commit
and ticket-in-branch (`feature/GN1-#####`): the blocking `commit-msg` hook is
**dropped** — spine derives the ticket instead of re-enforcing it (§5) — and a
**dashboard-at-scale** phase is added (§9). A **commit-trailer rule** binding
spine-written commits to their JIRA ticket is added (§5, extends `ADAPTER-CONTRACT.md` §6), and the
PR-creation decision is resolved (§6.2 — `/ship` opens a draft PR via a new
`open-pr` adapter, autonomy-aware, never auto-merging). Not yet built.

Origin: raised in a design session about scaling spine to any task size so
multiple engineering teams at the company (GolfNow One / G1) can adopt one
standard without heavyweight ceremony on small work. Two reference systems
were surveyed for transferable ideas — `~/society-of-mind` and the company's
own `~/g1-agent-tools` — cited by mechanism in §11. The user's core framing:
*"a standardized way to use Claude at the company that documents what needs
documenting, provides the proper checks for more consistent work, regardless of
task size/scope"* — while not flooding `docs/` with docs for small changes, and
making spine "almost idiot-proof" so it keeps getting used.

## The problem

Spine already has an adaptive-strictness dial — the Class 0/1/2 system — but
three things keep it from being a company-wide standard usable on small work:

1. **The bottom tier is invisible in-project.** Class 0 leaves no in-project
   trace; `scan-untracked-ratio` treats an untrailered commit as "Class 0 or
   off-spine work, by definition." The org's own ticket-in-commit rule makes
   work JIRA-traceable, but nothing in the *repo* records what spine did (or
   didn't) on a change. "Document what needs documenting regardless of size"
   has no in-project home at the low end.
2. **There is no front door.** `/task <description>` is the only entrance and it
   opens straight into classify -> research. No ticket ingestion, no
   ticket<->task linkage in spine's own state, no affordance for an engineer who
   doesn't remember the flow.
3. **The flow's friction is phase-shaped, not risk-shaped.** Even a clean Class 1
   bug fix stops the human three times (confirm class, approve plan, type
   `/verify`, type `/ship`). For a small ticket the stop-start rhythm — not the
   documentation — is what makes spine feel heavy, and heavy tools get routed
   around. Both surveyed systems are cautionary: `society-of-mind` reports ~14%
   real adherence to its own elaborate flow, and `g1-workflows` ships without a
   green-build gate at all. The lesson: a mandatory path that is too expensive,
   or only enforced in prose, produces theater — worse than no standard, because
   it manufactures false confidence.

## 1. The model everything follows from

**Two dials, deliberately separated.**

- **Dial A — human stop points.** How often the engineer is interrupted. Scales
  *down* as work gets easier. Costs the engineer's attention.
- **Dial B — verification depth.** Floor, adversary count, smoke, second
  approver. Stays *high* regardless of ease; scales only with genuine *risk*.
  Costs tokens/latency, not the engineer's time (except the second approver).

The whole design: **turn Dial A down for easy work; keep Dial B up.** Easy means
*fewer interruptions*, never *fewer checks*. The adversaries and floor run
without the human present, so they stay on even for an easy task; only the human
relay is removed.

**Two invariants keep this honest.**

1. **Blast radius caps autonomy — never the reverse.** The *class* (0/1/2) is the
   risk axis and sets a ceiling on how autonomous the flow may be. Class 2 can
   never run unattended, however small it looked. A mid-stream escalation
   (`core/hooks/path-escalate` firing when the real diff hits a protected path)
   revokes both class and autonomy in one move. This is what makes "call it easy"
   safe: reality can always take it back.
2. **Stops are exception-driven, not phase-driven.** The intake classification is
   a *revocable hypothesis*, so an easy task has no *scheduled* stops but the same
   *tripwires* armed as everything else — a halt-tier deviation, a mid-stream
   escalation, the 3rd-deviation circuit breaker, or a verify FAIL pulls the
   human in.

## 2. The ladder spine proposes

From below the pipeline, upward:

- **Not-a-task.** A question, read-only investigation, or throwaway spike with no
  commit. Spine recognizes it and *gets out of the way*: for a question, "this is
  a question, not a change — want me to just answer it? no task, no trace"; for a
  spike, "this is exploratory — spine is the wrong tool; work outside the pipeline
  and bring the result back as `/intake` once you know the shape" (consistent with
  `docs/tradeoffs.md`'s own "where this is worse than a bare agent"). Declining to
  engage is an adoption feature.
- **Class 0 — traced-trivial.** A real but trivial committed change (<=2 files,
  ~15 lines, no new public symbol, no protected path). Edit + one in-project
  trace line. No folder, no `docs/` file, no phases.
- **Class 1 — standard** (small bug/enhancement, the sweet spot). Runs the
  pipeline at autonomy `auto` / `checkpointed` / `guided`.
- **Class 2 — governed** (high blast radius). `guided` only.

Autonomy levels:

| Autonomy | Allowed for | Scheduled stops | Checks (Dial B) |
|---|---|---|---|
| traced | Class 0 only | none — just edit | none (trace only) |
| auto | Class 1 only | none; exception-stops only; human reviews finished PR | full floor + adversaries |
| checkpointed | Class 1 | one: plan approval, then a single "finish" action | full floor + adversaries |
| guided | Class 1 or 2 | every phase boundary; second approver on Class 2 | full, maximum |

The sharpest trade, stated up front: **`auto` removes the human plan-approval
checkpoint.** `docs/tradeoffs.md` already names plan review as "the one link in
the chain with no backstop." `auto` trades *pre-implementation* plan review for
*post-implementation* PR review, adversaries still running in between, blast
radius bounded to Class 1. Handling in §7.

**What the engineer actually gains, by tier (stated honestly).** Class 1 is where
the engineer gains value *for themselves on the task* — the adversaries catch what
they'd ship (the worked example's TOCTOU race, the missing test coverage), the
plan forces acceptance criteria, and `auto` delivers that without the stop-start.
Class 0 gains the engineer ~nothing directly; the *org* gains in-project
traceability at near-zero cost to them (§5). Not-a-task gains nothing by engaging,
so spine doesn't.

## 3. `/spine` — the front desk (idiot-proofing)

Read-only, human-typed (`disable-model-invocation: true`) command meaning "you
never have to remember how spine works." New `core/skills/spine/SKILL.md`.
State-dependent:

- **Install broken / half-wired** — runs `core/scripts/setup --check` itself and
  reports exactly what is wrong plus the one fix command. When hooks/symlinks are
  half-installed, `/spine` diagnoses instead of leaving the engineer on a bare
  denial. (The one state it cannot reach is a repo with no spine skills symlinked
  at all — there the committed `.claude/hook-guard` denial message is the pointer;
  update that message to name `/spine` as the next step once installed.)
- **Installed, nothing in progress** — a short menu, one sentence each,
  recommending the starting move: `/intake <TICKET>` (feed it a ticket; it sizes
  the work and proposes how to run it — the first step for any work), `/task
  <desc>` (start without a ticket), `/visualize` (project dashboard), `/tasks`,
  `/costs`.
- **Task in progress** — the status update: task ID, class + autonomy, current
  phase, unacknowledged flags, open deviations, whether it is waiting on the
  human and for exactly what, and the single next action to type. Read from
  `.spine/current-task` / `work/<id>/state` / `work/<id>/class` / `flags.json` /
  `deviations.md`.
- **Workspace / multi-repo** — orients to workspace root vs. member repo.

Plus a one-line addition to `core/templates/CLAUDE.md` (within its 60-line cap):
**"Lost? Run `/spine`."** The largest adoption lever in the plan: the engineer
memorizes nothing; the front desk always says where they are and what's next.

## 4. `/intake` — ticket front door + right-sizer

New `core/skills/intake/SKILL.md` (`disable-model-invocation: true`). Flow:

1. **Preflight** — `setup --check`; defer to `/spine` diagnosis if the install is
   broken.
2. **Fetch the ticket** via a new tracker-agnostic capability adapter,
   `.spine/adapters/ticket-fetch <KEY>` (§4.1). Absent adapter -> manual paste.
3. **Clarify in-session** — rewrite the ticket into a brief marking confirmed vs.
   inferred. No doc file.
4. **Grounded pre-scan** (bounded, like `/adopt`'s survey) — locate where the
   change lands, check `.spine/protected-paths.conf`, detect public-symbol /
   schema / auth / multi-repo signals. Required, not optional: a complexity
   estimate from ticket text alone is astrology. Stop when there is enough to
   classify.
5. **Complexity signal** — compute a proposed `(class, autonomy)` using an explicit
   rubric (§6). A proposal, never a silent verdict.
6. **Confidence-weighted menu** — near one-tap confirm when spine is confident; a
   real fork when it is torn (recommended vs. "scope it first" vs. "deeper than it
   looks"); and when confidence is *low*, spine *leads* with "let me scope this
   first" — a bounded research spike whose only output is a better classification.
   Escalate-up always offered. A downgrade below spine's recommendation is a
   recorded override, surfaced in `/costs`.
7. **Route** — trivial -> traced-trivial fast path (§5); Class 1/2 -> continue
   *directly* into the pipeline at the chosen autonomy (unified entrypoint, §4.2).

### 4.1 The `ticket-fetch` capability

Added to `core/ADAPTER-CONTRACT.md`. Core knows only the abstract ticket shape
(key, title, description, comments, links) and the branch/commit ticket-key
pattern. The G1 adapter wraps the existing `g1-jira-intake` skill's proven `acli`
-> Atlassian MCP -> manual-paste chain and knows the `feature/GN1-#####`
convention; another org writes its own; an absent adapter degrades to manual
paste. Nothing in `core/` names JIRA, `acli`, or any tracker — the same
stack-independence rule the capability adapters already follow.

### 4.2 Unified entrypoint (resolved fork)

`/intake` absorbs the classify step and flows straight into the shared pipeline
engine — no second typed command to start work. `/task` remains the ad-hoc,
no-ticket entrypoint into the same engine. This is human-initiated, so it does not
conflict with any `disable-model-invocation` gate; the only gates that stay
human-relayed by default are `/verify` and `/ship`, and even those are handled by
shared machinery rather than the disabled skills under `auto` (§6.1).

## 5. Traced-trivial (Class 0) — derive the ticket, don't enforce it

The org already enforces ticket linkage: no work happens without a ticket, every
commit carries the ticket number, and branches follow `feature/GN1-#####`. So
spine does **not** add a blocking hook to re-enforce what the org's VCS/JIRA
integration already guarantees — duplicating an existing gate is exactly the
redundant machinery `/ratchet` and spine's minimalism reject. (This supersedes an
earlier version of this proposal that added an enforcing `commit-msg` hook.)
Instead:

- Spine **derives** the ticket from the existing branch name / commit convention
  (cheap parse; the `ticket-fetch` adapter already knows the pattern). No new
  enforcement, no commit-path tax.
- A Class 0 change leaves a lightweight **in-project** trace — a single
  `core/scripts/ledger trace <ticket> "<line>"` entry carrying the spine-specific
  facts JIRA does not hold (class, that no flow ran, who/when). No work folder, no
  `docs/` file.

**Commit-trailer rule (new — extends `core/ADAPTER-CONTRACT.md` §6).** Spine's
existing commit convention carries only `Spine-Task: <task-id>` and `Spine-Bypass:
<reason>` — it has no rule tying a commit to its JIRA ticket. This proposal adds
one: every spine-written commit (both `/ship` and the traced-trivial path) also
carries `Spine-Ticket: <ticket-key>` whenever a ticket is available — derived from
`/intake` or the branch/commit convention — composing alongside `Spine-Task:`. It
is spine's own commit-writing convention, **not** a blocking hook (the org already
enforces ticket-in-commit): it gives `ledger` / the dashboard / `scan-untracked-
ratio` a consistent structured field and an explicit spine-task<->JIRA join,
rather than parsing whatever subject-prefix format the org uses. Off-ticket work
omits the trailer and is counted as before.

**Why an in-project trace at all, when JIRA + commit attachment already link
commit->ticket?** Two things the bare JIRA link does not cover: (1) it carries
spine-specific metadata JIRA will never hold — class, autonomy, whether
adversaries ran and what they found, floor result — which makes a change
*understandable in-repo*, not merely *attributable*; (2) it feeds the spine
dashboard (§9), which reads in-project state, not JIRA — without it, trivial work
is absent from the dashboard entirely. The trace is durable with the repo
(survives a tracker migration) and queryable with `git log` / the dashboard
without an Atlassian round-trip.

**Scope, honestly.** Spine traces Class 0 work that comes through its front door
(`/intake` or the traced-trivial path). A raw trivial edit committed without
touching spine stays JIRA-traceable via the org's existing enforcement, and spine
does not chase it with a blocking hook — chasing every commit is the friction that
kills adoption. `scan-untracked-ratio` reframes from "is there a ticket" (the org
guarantees that) to "did this go through a spine flow" — informational, not a gate.

## 6. Autonomy mechanics (the heart)

Sizing rubric (borrowing `g1-agent-tools`' complexity score and
`society-of-mind`'s Governed-tier checklist):

- Class 2 auto-trigger checklist (any -> Class 2 -> guided): auth/authz change,
  data migration, external/public contract, >=2 owned systems, irreversible
  release, or any `.spine/protected-paths.conf` match.
- Otherwise a 1–5 complexity score maps predicted-touch count and structural
  breadth (persistence + backend + UI, multi-repo) to Class 0/1 and a proposed
  autonomy.

Halt-tier definition, sharpened (borrowing g1's rigor calibration): a decision is
halt-worthy only if it **changes a business/security/contract outcome AND is
expensive to discover later**; anything a compiler, the floor, or code review
would surface anyway is `record-and-proceed`, not `halt`. This same rule is what
`auto` uses to decide what counts as an exception-stop. Recorded in
`core/templates/plan.md`'s latitude section (template prose, not `CLAUDE.md`).

### 6.1 The verify/ship independence resolution

Extract the verify and ship *machinery* — floor run, `researcher`/`falsifier`/
`security` subagents, `verdict-filter`, and commit/briefing/PR-description
assembly — into shared procedures that both the human-typed `/verify` and `/ship`
skills **and** the autonomous pipeline invoke. Then `auto` runs verification
inline **without invoking the `disable-model-invocation` skill and without
touching that flag**. Independence is fully preserved, because it never came from
the human typing the command — it comes from the adversaries being fresh, isolated
subagents. What `auto` removes is the human relay, which the engineer explicitly
delegated by choosing `auto` at intake.

`auto`'s endpoint is a **PR ready for review — it never auto-merges.** The ship
machinery produces the commit + briefing + PR; the human's single touchpoint is
reviewing and merging, with the written plan, adversary findings, floor result,
and any deviations attached. The final human decision (merge) is preserved, just
relocated from scattered phase-boundary stops to one PR review.

### 6.2 PR creation (new `open-pr` capability)

`/ship` today commits **locally** and writes `pr-description.md`, then
deliberately stops — "pushing or opening a PR is the human's call" (§5/§6 of
`core/skills/ship/SKILL.md`). This proposal changes that, autonomy-aware:

- **`auto` requires PR creation.** Its single human touchpoint is reviewing the
  finished PR, so `/ship` must push the ticket-derived branch (`feature/GN1-#####`,
  from §5's ticket derivation) and open a **draft** PR with `pr-description.md` as
  the body, then stop. This **reverses** the current "human opens the PR" stance
  for `auto` — disclosed, not silent.
- **`checkpointed` / `guided`** — opening a draft PR is offered too (the org opens a
  PR for every change; the body is already written; a draft carries no merge
  claim), **profile-gated**, default-on for a PR-driven shop, off to keep today's
  "I'll open my own PR" behavior.
- **Never auto-merge**, any mode. `auto` produces the draft; the human reviews,
  marks ready, and merges — the final decision stays human.
- **Host-agnostic via adapter.** Opening a PR is host-specific, so it goes through
  a new `.spine/adapters/open-pr` capability (abstract title/body/base/head -> URL)
  added to `core/ADAPTER-CONTRACT.md`; nothing in `core/` names a host tool. The G1
  adapter wraps the existing `g1-ship` skill, which already pushes and
  creates/updates a draft PR.

## 7. Gap handling (resolved)

- **`auto` widens the plan-adequacy gap.** Do not add a human stop back. Backstop
  mechanically: (1) permanent Class-1 cap; (2) the plan is still *written* and
  attached to the PR, so plan review happens at PR time — post-hoc, not absent;
  (3) the falsifier's stub-out probe is *mandatory* for `auto` — stubbing the
  feature and re-running tests directly catches the core plan-adequacy failure
  ("tests assert nothing"); (4) a team profile can disable `auto` outright; (5)
  `/costs` tracks `auto` tasks and their post-merge revert rate — if `auto`
  reverts exceed `checkpointed`, the data says lower the ceiling. Falsifiable exit
  condition.
- **Intake pre-scan can under-scope.** Do not chase a smarter pre-scan; spend the
  reliability budget on *revocation*. The guess is re-validated at two mechanical
  points with better information: after **research** (deeper than the pre-scan —
  bigger/riskier scope drops autonomy a notch and/or escalates class) and against
  the **real diff** (`path-escalate`, already mechanical). The intake guess is
  never the last word. The deliberately-misleading worked example (§10) proves
  revocation fires.
- **In-project trace vs. JIRA.** The trace is justified by spine-specific metadata
  + the dashboard feed, not by re-linking commit->ticket (JIRA owns that). Scoped
  to front-door work only, so it adds no commit-path cost and no redundant gate.
- **Profiles invite fragmentation.** Three defenses: (1) the hard-floor invariant
  is *mechanically enforced at profile load* — a profile that tries to disable the
  floor, the protected-path hook, or the autonomy cap is rejected the way
  `adapter-conformance` rejects a bad adapter; (2) ship only a few named presets
  (prototype / standard / regulated) with explicit overrides, not a wall of dials;
  (3) `/costs` reports per-team drift (trivial ratio, below-recommendation override
  ratio, bypass ratio). Repo-level only in v1.

## 8. Team strictness profiles

Committed `.spine/profile.json` (repo/team-level), read where thresholds are
currently hardcoded (Class 0 threshold, Class 1 adversary count, autonomy ceiling,
smoke-in-floor). Presets prototype / standard / regulated, overridable. Added to
`/adopt` and `/bootstrap` as a Layer-2.5 calibration step. Hard invariant per §7.

## 9. Dashboard at scale

`/visualize` (`core/scripts/render-dashboard`) today renders a Gantt of every
task, a chronological event feed, the decision store, the capability matrix,
milestone progress, and the drift instrument (see
`docs/proposals/task-visualization.md`, v1 IMPLEMENTED). With multiple engineers
over months, the flat timeline, event feed, and report list grow unbounded and
stop being usable. The whole point of this initiative — more adoption — makes this
worse, so it belongs in the same plan.

Requirements:

- **Default to a bounded window.** Landing view shows recent + open (e.g. last 14
  days plus all not-yet-shipped tasks), not the full history. A `--since`/`--all`
  flag (mirroring `/costs --since`) widens it. The self-contained HTML must not try
  to render months of events on load.
- **Facet + filter** by owner, class, autonomy, phase/status, milestone, ticket
  key, contract, and date range. Client-side (the file is static and
  CSP-sandboxed), over the windowed data set.
- **Search** free-text over task titles, ticket keys, decision IDs.
- **Group and collapse.** Tasks grouped by milestone / week / engineer; completed
  collapsed by default; the event feed grouped by day with per-day collapse. A
  landing summary (counts by class/status/engineer + drift metrics) with
  drill-down, rather than a wall of rows.
- **Bound the artifact.** Window server-side (render recent by default; `--all`
  regenerates a fuller file) or virtualize client-side; a hard size budget with a
  "covers <window>; use --since for older" banner so windowed-empty is never
  mistaken for nothing-happened.
- **Report index, not a flat list.** `/task-report` outputs get a searchable,
  grouped index (by milestone / engineer / date).

This is a v2 of the existing visualization proposal, which already flagged the
milestone swimlane as v2-open — it extends that rather than starting fresh.
Independent of the intake/autonomy work; can be built in parallel, and should land
before a team has run enough tasks to make the current dashboard unusable (~4–6
weeks of multi-engineer use).

## 10. Build order and validation

Ship independently, each phase validated with a real worked example ("the worked
example is your integration test"):

1. **`/spine` front desk** — read-only, low-risk, immediately useful, makes
   everything after it discoverable. The `CLAUDE.md` one-liner.
2. **Traced-trivial** — derive the ticket from branch/commit convention +
   `ledger trace` + `scan-untracked-ratio` reframe. No enforcement hook (§5).
3. **`/intake` + `ticket-fetch` adapter** — grounded pre-scan, confidence-weighted
   menu, complexity rubric.
4. **Autonomy mechanics** — extract shared verify/ship machinery; implement
   auto/checkpointed/guided; wire autonomy<=class; relocate stops per mode. Most
   invasive; depends on 3.
5. **Team profiles.**

Parallel, independent track:

- **Dashboard at scale (§9)** — depends on nothing above; land before ~4–6 weeks
  of multi-engineer use.

Sizing/halt sharpening (§6) folds into 3 and 4.

Worked examples for validation: a trivial ticket (`traced`), an easy bug (`auto` —
the key new path), a medium enhancement (`checkpointed`), a Class 2 change
(`guided`), and one **deliberately-misleading** ticket that looks easy but touches
a protected path — to prove mid-stream escalation fires and drops to `guided`.
Measure all with `/costs`.

## 11. What we borrow, and what we reject

From `~/g1-agent-tools` (highest justification — the company's own proven
standard): reuse `g1-jira-intake`'s ingestion chain behind the `ticket-fetch`
adapter; reuse `g1-ship`'s push + draft-PR creation behind the new `open-pr` adapter
(§6.2); adopt its complexity-score->auto-split rubric for sizing; adopt its
"blocking only if it changes an outcome AND is expensive to discover later" rigor
calibration for the halt tier. The integration is complementary: spine supplies
the real verify/ship gate `g1-workflows` explicitly lacks, and could receive a
handoff from `g1-workflow-handoff` at the verify/ship boundary (optional, out of
v1 scope).

From `~/society-of-mind`: the self-deception-resistant depth gate — the label that
unlocks skipping rigor is checklist-gated and re-validated, never a silent
self-grade (the shape of §4 step 6 + §7 revocation); and its Governed auto-trigger
checklist (§6).

Rejected as already-covered-or-better in spine: SoM's 35-agent society and 9-gate
flow (spine's 3 agents + floor cover it; SoM's ~14% adherence argues against the
elaboration); `second-guess`/`prove-it` (spine's falsifier + security + the
"could-not-run != passed" tooling-gap discipline already embody "BLOCKED over
degraded"); the `.out-of-scope` KB (spine has decision records + `DEFERRED.md`);
`g1-workflows`' LangGraph runtime (spine's skills-as-instructions model is
deliberate and portable, and spine's real floor is better than g1's missing one).

## 12. Conceded gaps / open questions

- **`auto` plan-adequacy** — mitigated (§7), not eliminated. The mandatory
  falsifier stub-check and PR-time plan review are partial backstops; a plan with
  subtly wrong acceptance criteria can still pass. Class-1 cap bounds the damage.
- **Pre-scan under-scoping** relies on revocation firing; if both research and
  `path-escalate` miss a semantic (not textual) risk, `auto` proceeds. Same class
  as the existing "semantic collisions between sequential tasks" gap.
- **In-project trace scope** — spine only traces front-door work; raw trivial
  commits are JIRA-traced only. Accepted deliberately, to avoid commit-path
  friction and a gate redundant with the org's existing enforcement.
- **Unified entrypoint** is a larger refactor of the front of `/task` than the
  fallback (intake records, human types `/task`); chosen for the low-friction
  goal, cost disclosed.
- **PR-creation reverses `/ship`'s current "human opens the PR" stance** (§6.2).
  Necessary for `auto`; profile-gated for the rest, so a team can keep the old
  behavior. A draft PR never implies merge-readiness.
- **Profiles are repo-level only.** Multiple teams sharing one repo at different
  strictness is unsupported in v1.
- **Dashboard windowing** makes older history a deliberate second click; the
  "covers <window>" banner prevents mistaking windowed-empty for truly-empty.
