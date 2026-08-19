# Proposal: routing flagged-but-unfixed adversary findings into `milestone.md`

Status: IMPLEMENTED — 2026-08-19. Confirmed with the user, including the
§2a dedup refinement and §6 lifecycle removal (both folded in below —
§6 was reclassified from "deferred follow-up" to "build now" mid-session:
its mechanism is small, mirrors an existing pattern (§2's decision-status-
flip) exactly, and degrades safely if wrong, so there was no real reason
to wait for a worked example that validation alone needed). Spine has no
`docs/decisions/`-style store or `/task` harness pointed at its own
checkout (this repo has no `.spine/`; it is the portable core, never a
bootstrapped project), so this proposal, confirmed by the user, was the
whole review process for the change — the same shape `docs/proposals/
intake-and-adaptive-autonomy.md` already used for its own confirmation.
Landed in `core/ADAPTER-CONTRACT.md §5` (the `disposition` field),
`core/skills/verify/SKILL.md §5` (writes it), `core/templates/milestone.md`
(the `## Known gaps` section and its `next-gap-id` counter),
`core/templates/plan.md` (the `## Resolves known gaps` section),
`core/skills/task/SKILL.md §3` (writes it), `core/skills/ship/SKILL.md`
(§3 split into §3a/§3b/§3c), and `core/templates/briefing.md`/
`core/templates/pr-description.md` (the `Milestone` sections that surface
§3a's and §3c's results). All six build-order steps are done.

Origin: surfaced by a real task, not hypothesized. `20260818-supabase-schema-verdicts`
(member task 2 of `work/M0`) ran two full adversary rounds; three kept,
not-fixed findings were real, cross-task-relevant gaps deliberately left
for whichever future `M0` member task builds the thing each one blocks
(`work/20260818-supabase-schema-verdicts/verify.md`'s "Adversary verdicts"
section; the fix-vs-flag reasoning for each is in that task's `notes.md`,
"Adversary findings fixed vs. flagged"). Nothing in spine prompted routing
those findings anywhere task 3/4 would see them at planning time — the
user hand-added a "## Known gaps for future member tasks" section to
`work/M0/milestone.md` after `/verify` completed, because `milestone.md` is
the one place a future member task's planning context is guaranteed to
load it (`core/skills/task/SKILL.md`'s `--milestone <id>` handling, "before
classifying"). That hand-add is the shape of the fix; this proposal is how
to make it mechanical.

## The gap, precisely

`core/skills/ship/SKILL.md` §3 ("Milestone done-definition") reads
`milestone.md`'s `## Member tasks` list and checks two things: is this the
milestone's completing ship, and if so, are the named capability targets
actually `implemented`. It never reads `work/<task-id>/verify.md`'s
Adversary verdicts at all. §4 ("Write the delta briefing") does surface
adversary findings — "count + max severity + one-line gist each, pointer
to `verify.md`, never compressed further" — but that's the **per-task**
summary a human reads once, for this task's own PR review. A future
member task's planning context never re-reads a past sibling's briefing;
it reads `milestone.md` (`core/skills/task/SKILL.md`, `--milestone`
handling). A finding that lives only in `verify.md`/`notes.md`/`briefing.md`
is therefore fully documented and fully invisible to the one session that
would actually need it — task 3 or 4's own `/task --milestone M0` planning
context has no reason to open a sibling task's `work/` folder on its own.

This is the same failure shape spine already names and guards against
elsewhere: `core/ADAPTER-CONTRACT.md`'s "never silently skip a gate"
principle (quoted verbatim at `core/skills/verify/SKILL.md`'s multiple
tooling-gap sections), and the entire reason `verify.md` is assembled
mechanically from script output instead of trusted as an agent's own chat
summary (`core/templates/verify.md`'s own header comment: "this file
quotes scripts, it doesn't paraphrase them"). A gap that is
adversary-confirmed, real, and recorded — but reachable only by a human
who happens to go looking in the right sibling task's folder — is
"technically recorded, practically silent," the exact phrase build prompt
§1 uses for the worst object this system can produce.

## The tension this has to hold, not skip past

`milestone.md` is loaded into **every** future member task's planning
context (`core/skills/task/SKILL.md`, `--milestone` step, "before
classifying"). Auto-writing every kept-not-fixed finding there would
recreate the exact cost `core/templates/CLAUDE.md`'s own editing rule
warns about — "every line here taxes every turn, forever" — one layer
down (every line here taxes every future member task's planning turn, for
the milestone's remaining lifetime). The real worked example proves this
isn't hypothetical: of the findings this task flagged, one — a low-severity
note that the plan's prose overstated its own DB-constraint mechanism
(`notes.md`'s "security's low S5 (round 1)") — was correctly judged, on
inspection, not a real gap, and correctly excluded from `milestone.md`.
An auto-append would have written it there anyway; a human had to look at
it and say no. This needs a triage step with real judgment in the loop,
not a mechanical auto-promote-on-severity rule.

## Model

### 1. A mechanical fixed/not-fixed field, not prose-parsing

Today, whether a kept adversary verdict was ultimately fixed lives only as
free prose in `verify.md`'s bullets ("— fixed.", "— flagged, not fixed.",
"— still open, unaddressed…"). That prose is exactly what a human should
keep reading for *why* — but it is not something `/ship` should re-parse
to decide *whether to ask*; matching on "flagged" vs. "still open" vs.
"reported... but tested against stale state" (three different phrasings
for three different outcomes in the one real example this proposal is
grounded on) is fragile in exactly the way spine already rejects for
adversary output (`ADAPTER-CONTRACT.md §5`'s whole point is a
machine-checkable evidence shape, not a trusted narrative).

Add one field, `"disposition": "fixed" | "not_fixed"`, to each kept
verdict object in `work/<task-id>/artifacts/<agent>-verdict.json` — the
same file `/verify` already owns end to end. Write it at the exact moment
`/verify` §5 already makes this call in prose (deciding how to word each
bullet in `verify.md`'s Adversary verdicts section) — no new judgment, one
more field recording a decision already being made. `verdict-filter`'s
schema (`core/ADAPTER-CONTRACT.md §5`) gains this as an optional field
adversaries never populate themselves (they don't know what got fixed
after they returned); `/verify` populates it during assembly, defaulting
`not_fixed` for anything not explicitly marked fixed, so a missed field
never silently reads as resolved.

### 2. The trigger: any severity, not medium+-only

The natural first instinct — surface the question only when a not-fixed
finding is medium or higher — doesn't survive contact with this proposal's
own worked example. Of the three findings the user actually carried into
`milestone.md`, **two were low severity**: the `decided_at` lower-bound gap
and the `verdicts_current` tie-break gap (`notes.md`'s "Adversary findings
fixed vs. flagged", both explicitly `low`). A medium+-only trigger would
have silently never asked about either. Meanwhile the one low-severity
finding correctly *not* carried (the CHECK-constraint one) shows severity
alone doesn't predict relevance either direction.

So: the prompt fires whenever a milestone member task ships with **any**
`not_fixed`-disposition kept verdict, any severity, count > 0 — not
gated by severity. What severity does gate is *cost*, not *whether to
ask*: the question is a single batched prompt (§3 below), not one
interruption per finding, and it only fires once, at this task's own ship,
never repeated. Zero not-fixed findings (the common case — most adversary
findings get fixed) means the step is silently a no-op, same "never
silently skip a gate, but a gate that correctly never applied is not a
gap" discipline `core/skills/verify/SKILL.md`'s UI-render section already
uses.

### 2a. Dedup against gaps `milestone.md` already tracks

Two sibling member tasks can independently trip the same underlying gap —
e.g. both task 2 and task 3 could plausibly have their own adversary
rounds notice "`service_role` has no grants," from different angles, if
task 3 also touches the write path before task 4 resolves it. Asking the
human to re-triage a gap `milestone.md` already carries (§5) is pure
noise, and noise here is exactly what erodes trust in the prompt the same
way an over-firing lint rule erodes trust in lint.

Before building §3a's batched question, filter out any `not_fixed`
finding whose `claim`/`evidence.file` substantially matches an existing
`gap-<n>` entry's `source` field in `milestone.md`'s `## Known gaps`
section (cheap containment/substring check against the machine-fenced
block, not semantic matching — same fidelity level `check-stale`'s own
file-drift comparison already uses, no new infra). A match is recorded in
`notes.md` as "already tracked as gap-<n>, not re-asked" rather than
silently dropped — same "record the decision, not just the outcome"
discipline §3 below applies to a declined finding, since a suppressed
question is its own kind of decision. A near-miss that the substring check
doesn't catch is not a regression — today it isn't tracked at all, so
worst case is one avoidable repeat question, never a missed one.

### 3. Where it lives: new `/ship` §3a, every member-task ship

`core/skills/ship/SKILL.md`'s current §3 only fires its check on the
**completing** ship of a milestone (every member task done). This gap is
different in shape: task 2 shipped with real, milestone-relevant findings
while tasks 3/4 were still `TBD` — the completing-ship-only trigger would
never have caught it. Split §3 into:

- **§3a "Flagged-finding triage"** — runs on **every** member-task ship
  (`work/<task-id>/milestone` set at all, same precondition §3 already
  checks, just not gated on "is this the last one"). Reads this task's own
  final `work/<task-id>/artifacts/<agent>-verdict.json` for each adversary
  that ran, filters `disposition == "not_fixed"`, then applies §2a's dedup
  filter against `milestone.md`'s existing `## Known gaps` entries. Zero
  survivors (either none were flagged, or every flagged one is already
  tracked): nothing to do, no section in the briefing. One or more
  survivors: batched question (autonomy-dependent, §4 below), each option
  showing severity + claim + evidence pointer, per `AskUserQuestion`-style
  multiSelect ("which of these should carry into `milestone.md`'s Known
  gaps for future member tasks to see? none is a valid answer"). Whatever
  the human picks, **record the decision, not just the outcome** — every
  not-fixed finding (deduped or not) gets a line in `notes.md`: carried
  (with the new gap's id, §5), declined (one reason, even if just "not
  real, see verify.md"), or already-tracked (§2a's outcome, with the
  existing gap's id). A finding the human looked at and declined must read
  differently, permanently, from a finding nobody ever asked about — the
  same "silently treat as pass" failure `core/skills/verify/SKILL.md`'s
  tooling-gap discipline names for scripts applies here to findings.
- **§3b "Milestone done-definition"** — unchanged, still completing-ship-
  only, still what it is today.

### 4. Autonomy interaction

`core/skills/task/SKILL.md` §5 already branches `/ship` on
`work/<task-id>/autonomy`. §3a has to branch the same way, since the
underlying mechanism — an interactive multi-select question — assumes a
human is present:

- **`guided`** (a human is literally the one typing `/ship`): ask
  interactively, block until answered, same posture as every other
  `guided` stop. Apply the carried entries to `milestone.md` before the
  commit in §5, same turn.
- **`checkpointed` / `auto`** (no scheduled stop at ship time): never
  block on this — it is explicitly not a merge-gate concern (adversary
  findings never fail `/verify` by that skill's own §6, and this proposal
  doesn't change that). Instead, draft the candidate carry-forward entries
  (§5's shape) and place them in the briefing/PR description under a
  clearly labeled "Proposed milestone gap entries — undecided" section,
  **not applied to `milestone.md`**. The human's post-hoc PR review (the
  same relocated touchpoint `docs/proposals/intake-and-adaptive-autonomy.md`
  §6.1 already establishes for `auto`'s plan review) is where they get
  triaged — carried by hand-editing `milestone.md`, or left. This keeps
  the same invariant §1's mechanical field exists to protect: nothing
  gets written to the always-loaded `milestone.md` without a human having
  looked at it, whether that look happens synchronously (`guided`) or at
  PR review (`checkpointed`/`auto`).

### 5. The `milestone.md` addition

Add a `## Known gaps for future member tasks` section to
`core/templates/milestone.md`, positioned after `## Inter-task contracts`
and before `## Capability targets` — the same position the user's
hand-written version already used in `work/M0/milestone.md`. Machine-fenced
like `plan.md`'s `## Predicted touch` (`core/templates/plan.md`'s own
convention, mechanically parsed by `core/scripts/conformance`), one entry
per carried finding, a stable `gap-<n>` id so a later task's `/ship` can
remove a specific entry by id rather than fuzzy-matching prose:

```
## Known gaps for future member tasks

<!-- MACHINE: known-gaps -->
- id: gap-1
  source: work/20260818-supabase-schema-verdicts/verify.md (security, medium)
  **`service_role` has no grants on any table.** Safe today — no
  privileged path exists yet to be blocked — but the first privileged
  server-side write (task 3's session creation, or task 4's write path)
  gets `permission denied` until someone decides `BYPASSRLS` vs. per-table
  `for all using (true)` policies.
<!-- /MACHINE -->
```

`/ship`'s §3a drafts this prose from the verdict's own `claim`/`evidence`
plus `milestone.md`'s `## Member tasks` list (already loaded for §3b) so
it can name which future member task plausibly closes the gap, the same
way the real example does — never copied verbatim from `verify.md`'s
adversary-voice prose, since that voice is written for an attacker's
audience, not a future planner's.

### 6. Lifecycle: a gap that gets closed should leave — **Implemented**

A monotonically-growing `## Known gaps` section recreates exactly the tax
this proposal's own tension section (above) worries about, just deferred.
Built as a new, dedicated `## Resolves known gaps` section on `plan.md`
(`core/templates/plan.md`), mirroring `## Grounds on decisions`'s own
optional-appendix shape exactly rather than overloading `## The gist`'s
free prose or forcing every resolution through a distilled decision record
— a gap resolution is its own citation, not always a decision-worthy one.
`core/skills/task/SKILL.md` §3 tells the plan-writing step to add it when
a milestone's listed gap is actually being closed. `core/skills/ship/
SKILL.md` §3c reads it and removes the matching fenced entry from
`milestone.md` by id — mirrors §2's existing decision-status-flip
mechanic (find by id, edit exactly that one thing, nothing else in the
file changes). A gap resolved implicitly, with no explicit citation, is
not auto-detected — same honest-limit posture §0's ship-time re-grounding
already takes ("discovered late… is real, mechanical, and specific… not
implied by silence" applies in the other direction here too: silence about
a gap doesn't mean it's gone).

**A real correctness bug caught while implementing this, not while
designing it**: the original id-allocation instruction ("next id = one
more than the highest existing `gap-<n>`") breaks the moment removal
exists — remove the current highest-numbered entry and the next
allocation reuses its id for something unrelated, which would make a past
task's plan.md citation of "gap-3" permanently ambiguous between two
different findings. Fixed by adding `next-gap-id`, a monotonic counter
living in `milestone.md`'s own `## Known gaps` fence (`core/templates/
milestone.md`), incremented on every allocation, never decremented on
removal. Worth naming explicitly: this is exactly the kind of thing a
worked-example validation (§ Build order, step 5) doesn't catch by
itself — the grounding example never removed anything, so this bug was
invisible to that trace and only surfaced by reasoning through the
mechanism's own steady-state behavior once it existed as real
instructions, not prose describing an intent.

### 7. Explicitly out of scope

- **Tasks outside a milestone.** `work/<task-id>/milestone` absent means
  §3a skips entirely, identical to §3b's existing precondition. A flagged
  finding on a non-milestone task has no cross-task home under this
  proposal — same limitation the existing done-definition check already
  accepts, not a new gap this proposal introduces. Worth a future
  proposal (perhaps a repo-level "orphan gaps" ledger analogous to
  `/design`'s `docs/decisions/DEFERRED.md`) but conflating adversary
  findings with foundational-decision deferrals would be scope creep here.
- **Multi-repo.** `core/templates/verify.md`'s Adversary verdicts section
  is already task-scoped, not per-repo (unlike Floor results/Contract
  conformance, which repeat once per edited repo) — it already covers the
  full multi-repo blast radius in one place. `milestone.md` itself has no
  documented multi-repo shape today (its template is silent on
  `workspace.json`). This proposal makes no new multi-repo claim: §3a reads
  the one task-scoped verdict file exactly as `/ship` §3b already reads
  the one workspace-root `milestone.md`, so it inherits whatever
  single-workspace-root assumption already exists rather than resolving
  it.

## Alternatives considered and rejected

- **Auto-append every not-fixed finding, no triage.** Rejected outright —
  the CHECK-constraint counterexample above is real, not hypothetical;
  this would have polluted `milestone.md` on this proposal's own grounding
  example.
- **Medium+ severity as the trigger** (the shape suggested at the top of
  this investigation). Rejected — checked against the actual worked
  example, it misses 2 of the 3 findings a human really did want carried.
  Severity still matters for *how loudly* something reads once shown, not
  for *whether it's shown at all*.
- **Parse `verify.md` prose for "flagged"/"still open"/etc.** Rejected —
  the real example alone uses at least three different phrasings for
  "not fixed" and one ("reported as still-open… tested against stale
  state… now closed") that reads like "not fixed" but means the opposite.
  A mechanical field written at the same moment the prose is written costs
  one line and removes the ambiguity entirely.
- **Fire only on the milestone's completing ship** (matching existing
  §3's trigger). Rejected — the grounding example's findings surfaced on
  member task 2 of 4; waiting for task 4 to ship would mean task 3's own
  planning never saw them, defeating the purpose.

## Build order

1. **Done.** `core/ADAPTER-CONTRACT.md §5` — added the optional
   `disposition` field to the verdict schema (additive, not a breaking
   change to the existing shape; `verdict-filter` needed no code change
   since its `kept`/`dropped` selection already passes each verdict object
   through unmodified, field-agnostic).
2. **Done.** `core/skills/verify/SKILL.md` §5 — writes `disposition`
   alongside the existing prose bullet, same assembly step.
3. **Done.** `core/templates/milestone.md` — added the `## Known gaps`
   section and its machine fence.
4. **Done.** `core/skills/ship/SKILL.md` — split §3 into §3a (new,
   including §2a's dedup), §3b (renamed, unchanged logic), and §3c (new,
   §6's lifecycle removal — see item 6 below). Wired the autonomy branch
   (§4 above). Also updated `core/templates/briefing.md` and
   `core/templates/pr-description.md`'s own `Milestone` sections so §3a's
   and §3c's results — carried entries, the drafted undecided list for
   `checkpointed`/`auto`, and any gaps this task's own plan closed out —
   are visible to whichever artifact a task's human touchpoint actually
   reads.
5. **Done — hand-traced.** `20260818-supabase-schema-verdicts` is Class 2,
   `autonomy` file absent (`guided`), `milestone` = `M0`, already shipped —
   so this is a hand-trace against its recorded artifacts, not a live
   re-run. §3a's gather step, applied to the task's final round-2
   `falsifier-verdict.json`/`security-verdict.json`, surfaces 5 kept
   verdicts that would carry `disposition: not_fixed`: falsifier's
   `service_role` (medium), falsifier's D-6 CHECK-constraint note (low),
   falsifier's `verdicts_current` tie-break (low), security's `service_role`
   (low — the same underlying gap as falsifier's, raised independently by
   the other agent), and security's `decided_at` lower-bound note (low).
   `milestone.md`'s `## Known gaps` is empty at this point (M0's first
   ship) so §2a's dedup filter passes all 5 through unchanged. Presented
   as one `guided` batched question, a human triaging these reaches the
   same three entries actually written by hand — merging the two
   `service_role` verdicts into one `gap-1` (the mechanism doesn't
   auto-merge same-gap verdicts raised by different agents in the same
   ship; a human naturally does when writing the carried entry, same as
   the real one did) — plus `gap-2` (`decided_at` bound) and `gap-3`
   (tie-break), declining the D-6 CHECK-constraint one with the same
   reason `notes.md` already records ("on inspection, not a functional
   gap"). Reproduces the real outcome; the one nuance worth naming for a
   future reader is that "5 raw candidates → 3 final entries" is expected
   whenever more than one adversary independently flags the same
   underlying issue, not a bug in the gather step.
6. **Done — built, not yet worked-example-validated.** Lifecycle removal
   (§6): `core/templates/plan.md`'s `## Resolves known gaps` section,
   `core/skills/task/SKILL.md §3`'s instruction to write it, and
   `core/skills/ship/SKILL.md §3c`'s removal-by-id logic, plus the
   `next-gap-id` monotonic-counter fix §6 now documents. Reclassified from
   "deferred follow-up" to "build now" mid-session — the mechanism is
   small and mirrors §2's existing pattern exactly, and its failure mode if
   subtly wrong is silent-inaction (an entry that should've been removed
   stays listed, i.e. today's behavior), never a silent break — so waiting
   for `M0` to actually produce a gap-closing task bought nothing but risk
   of the follow-up never happening. Still genuinely untested against a
   real task, unlike step 5's hand-trace: the first real member task that
   cites `## Resolves known gaps` is this mechanism's actual integration
   test, and is worth a deliberate look when it happens, not just trust.

## Conceded gaps

- **Non-milestone flagged findings stay exactly as invisible as they are
  today** (§7) — this proposal narrows scope to the milestone case the
  real example demonstrated, not "every cross-task-relevant finding
  anywhere."
- **The drafted prose in §5's entries is model-written**, same trust level
  as every other spine artifact a model drafts and a human approves — a
  bad draft is a bad draft the human catches at the triage prompt (`guided`)
  or PR review (`checkpointed`/`auto`), not a new unchecked surface.
- **Lifecycle removal (§6) depends on a future task explicitly citing the
  gap id.** An implicit resolution — someone fixes the underlying issue
  without realizing it closes a listed gap — leaves a stale entry until a
  human notices. No mechanical staleness check is proposed here; if this
  turns out to matter in practice, it's a `/ratchet`-eligible finding once
  it's recurred, not something to preempt now.
