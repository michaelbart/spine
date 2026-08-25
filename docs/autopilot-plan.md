# Implementation plan: `/autopilot` — unattended, whole-roadmap execution

Status: proposed, not built. Experimental by design — explicitly not meant
to become spine's default posture. Written 2026-08-25, after confirming
the gap and two decisions from the human:

- **Class 2 escalations do not stop the run.** Autopilot proceeds on its
  own judgment even through protected-path/schema/auth changes and the
  second-approver requirement, logging every instance for a single
  end-of-run review — not a per-task one.
- **No per-task draft PR.** Commit and push directly; skip `open-pr`
  entirely.

## What exists today, and the actual gap

`/task` with an empty description already auto-continues to the next
queued milestone task (`core/skills/task/SKILL.md`'s header, backed by
`core/scripts/next-milestone-task`) — but only when a human types `/task`
again. Class-1 `auto` autonomy already removes the plan-approval stop and
runs verify/ship inline with no scheduled stop for *that one task*
(`core/skills/task/SKILL.md` §3/§5) — code is committed and pushed before
the draft PR even opens, so the PR is a post-hoc review artifact today,
not a merge gate. What doesn't exist anywhere: something that keeps
calling `/task` after each ship, all the way through a whole backlog of
milestone tasks, with zero human involvement — and, per what's being
asked here, that also removes the *other* stops `auto` autonomy currently
leaves in place: Class 2's forced `guided` + second-approver, halt-tier
deviations, the circuit breaker's "tell the human plainly," claims-check
blocks, and flag-blocked advances.

## The one invariant this plan does not touch

Spine's own autonomy dial already draws this line: **"checks (floor,
adversaries) never scale down with autonomy; only stops do"**
(`core/skills/task/SKILL.md`, the `autonomy` field's own description).
Autopilot is the same dial pushed to its extreme point — every remaining
*stop* becomes "decide, log, continue" — but every *check* stays real:
the deterministic floor, the falsifier's stub-out probe, the security
adversary, `claims-check`, `conformance`, `contract-touch` all still run
for real and can still fail a task outright. This is the whole reason the
experiment is survivable: nothing here proposes skipping verification,
only skipping the human being asked to bless a decision before or after
it. Git history stays the actual safety net — every commit is real, every
`deviations.md`/ledger record stays honest, so a human reviewing after
the fact can always read exactly what happened and revert anything wrong.

## Design

### Where it lives: a separate skill, not edits to `/task`/`/verify`/`/ship`

Recommend building `core/skills/autopilot/SKILL.md` as a **self-contained**
skill that does not modify `task/SKILL.md`, `verify/SKILL.md`, or
`ship/SKILL.md` at all — it re-runs the same underlying scripts
(`ledger`, `registry-sync`, `claims-check`, `conformance`,
`next-milestone-task`, the `researcher`/`falsifier`/`security` agents)
directly, with the override behavior written into its own procedure from
the start, rather than threading an `if autopilot-mode` branch through
three files that are also the primary, human-facing product. Tradeoff,
named explicitly: this duplicates a fair amount of `/task`'s phase logic
rather than reusing it in place. Worth it here because (a) this feature
is disclosed as possibly-temporary and (b) it keeps zero risk of an
autopilot-only code path accidentally firing during a normal `guided`
task. If it graduates from experiment to real feature later, that's the
point to fold the two back together.

### Preconditions (checked once, at the top)

- At least one `work/M*/milestone.md` exists — this is explicitly "you've
  already got milestones/plan/product spec, now go," not a replacement
  for `/roadmap`. If none exist, say so and point at `/roadmap` (or
  `/wayfinder` for a foggier starting point) instead of guessing at
  milestone boundaries itself. Auto-planning a *new* milestone from
  `docs/vision.md` when the current one finishes is out of scope for this
  pass — flagged below as a real open question, not assumed.
- The same two "can't self-resolve, not a judgment call" checks `/task`
  already runs stay hard stops here too, unchanged: core version
  `mismatch-strict` (Step 0), and missing git identity
  (`git config user.name`/`user.email`). Neither is a design decision
  autopilot could reasonably render a verdict on — one says this
  machine's tooling is out of sync, the other says spine has nothing to
  attribute authorship to. Both remain genuine stops.

### The loop

```
loop:
  next-milestone-task --project <root>
    TBD <id> <desc>   -> self-confirm ("autopilot: continuing <id>: <desc>"),
                          log it, run the task (below)
    BLOCKED <id> <tok> -> nothing runnable; end the loop (not an override —
                          there's no decision here, just unmet sequencing)
    WAITING <id>       -> same as BLOCKED; end the loop
    NONE                -> every milestone complete (or none exist); end the loop
    could not run        -> tooling gap; log it and end the loop (this is
                          infrastructure failing, not a design call to render
                          a verdict on)
  run one task to completion (classify -> research -> plan -> implement ->
  verify -> ship), inline, autonomy fixed at the autopilot level (see
  "What each stop becomes" below)
  commit + push directly; no open-pr call at all
  go to top of loop
```

### What each of `/task`'s stops becomes

Every row: what happens instead of stopping, and what gets appended to
`.spine/autopilot-log.md` (new, project-root, accumulates across the
whole run — not scoped inside one `work/<task-id>/`, since it has to
survive many tasks).

| Today's stop | Autopilot's replacement |
|---|---|
| Class confirmation | Use the suggested class as computed; log the class and why. |
| Autonomy selection | Fixed for the whole run — no per-task choice. |
| Plan approval | Skip, same as today's `auto` — write the plan, don't wait. |
| Class-2 forced `guided` | Overridden: proceed as if `auto`. Log every occurrence — this is the one explicitly called out for the end-of-run review. |
| Class-2 second approver | Self-approve via the *existing* override path (`approval.json` `"override": true`, `"override_reason": "autopilot: unattended run"`) — reusing the mechanism `/task` already has for a genuine solo/vacation-coverage situation, not inventing a new one. Log it. |
| `claims-check` block | Take the "override" resolution path already defined in `/task` §3, log the conflicting task/owner. |
| Halt-tier deviation | Decide on its own best judgment, write the `deviations.md` record with tier `halt` but resolve it immediately (status `resolved`, resolution = its own reasoning, tagged `autopilot`). Log it. |
| Circuit breaker (3 deviations) | Not actually a stop even today in spirit — reset to research and keep going inline (research needs no approval; plan approval is already skipped). Log the reset and why. |
| Flag-blocked advance | Read the flag, decide whether grounding still holds, acknowledge it (`acknowledged_by: "autopilot"`), log the reasoning either way. |
| `verify` FAIL | Already self-healing under today's `auto` (fix, re-verify inline) — keep that, but see the runaway guard below for when a task just won't pass. |
| New-milestone-creation confirmation | Self-confirm "proceed." Log it. (Expected to be rare in this pass, since milestones are a precondition, not something autopilot invents.) |

### A guard worth flagging, not silently building in

Nothing above bounds how many times a single task can bounce through
fix → re-verify → circuit-breaker-reset before it's actually stuck rather
than making progress — "no stopping" plus a genuinely broken task could
spin forever. Suggest capping it (e.g., two circuit-breaker resets on the
same task — six total deviations — is treated as "this task could not be
completed unattended," logged prominently as a real failure rather than a
confident decision, and the loop moves on to whatever's next). This is my
addition, not something asked for — flagging it rather than quietly
including it, since it changes "figure it out on its own, always" into
"figure it out on its own, with one bailout valve." Worth a explicit yes/no
before it's built.

### The end-of-run report

Whenever the loop ends (`NONE`, `BLOCKED`/`WAITING`, a tooling gap, or a
real stop like `mismatch-strict`), render `.spine/autopilot-log.md` into a
plain-language summary and say it out loud, not just leave it as a file:
tasks shipped (ids + one-line titles), and every logged override grouped
by kind (Class-2 escalations proceeded through, second-approver
self-approvals, halt-tier deviations self-resolved, claims-check
overrides, circuit-breaker resets, flags self-acknowledged) — each with
the task id, a one-line "what it decided," and enough of a pointer
(`deviations.md`/`approval.json`/etc.) that a human can actually verify
it, per the ask. This is the single human touchpoint the whole feature
has: after everything, not during.

## Open questions to resolve before/while building

1. **Does the run ever plan a *new* milestone itself** once the current
   one finishes (reading `docs/vision.md` the way `/spine`'s idle report
   already points a human at), or does it always stop at `NONE`? Left out
   of this pass on purpose — deciding milestone boundaries from a product
   spec is a materially different (and harder) kind of judgment call than
   the implementation-level ones above.
2. **The runaway guard's exact threshold** (see above) — needs a yes/no
   and a number, not just "some cap."
3. **Where autopilot itself gets invoked from** — a plain `/autopilot`
   with no arguments (simplest), or one that also accepts a milestone id
   to scope to (`/autopilot --milestone M3`) for a smaller first trial
   than "the whole backlog." Recommend supporting the scoped form from
   the start — a first real test almost certainly wants to bound the
   blast radius to one milestone, not the entire roadmap, the first time
   this runs for real.

## Suggested build order

1. `.spine/autopilot-log.md` shape + the end-of-run renderer, tested
   standalone against a handful of hand-written fake log entries before
   any real task runs through it — cheapest way to validate the report
   people actually see is legible before the loop that feeds it exists.
2. The loop skeleton (`next-milestone-task` → run one task → repeat),
   wired to today's *existing* `auto`-autonomy inline procedure verbatim
   (no new overrides yet) — proves the chaining works before adding the
   riskier part.
3. Layer in the Class-2/second-approver/claims-check/deviation/flag
   overrides from the table above, one at a time, each logged.
4. The runaway guard, once its threshold is settled (open question 2).
5. First real trial: scope to one small, already-planned milestone
   (`--milestone` from open question 3) in a real installed project, not
   this core repo — read the end-of-run report in full afterward and
   check every logged override against the actual commits before trusting
   a second, larger run.
