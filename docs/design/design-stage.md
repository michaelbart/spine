# The design stage and the M0 bootstrap waiver

Design rationale moved out of `docs/tradeoffs.md`. Describes `/design` and the one narrow waiver of the Class 2 smoke hard gate.

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

## The M0 bootstrap waiver

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

