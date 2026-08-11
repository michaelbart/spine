# Extension C, Phase D — two-engineer demo, solo-regression, tradeoffs, self-red-team, delivery

## Built

| File | Status |
|---|---|
| `core/scripts/second-approver-check` | New (promoted from Phase C's hand-verified prose logic — the self-red-team sweep found it was the one gate with zero script backing at all). |
| `core/scripts/claims-check` | Extended: `--diff <base-ref>` mode — real-diff-vs-other-open-tasks' claims, tagging `[UNDECLARED]`/`[DECLARED]`. Closes the narrow-claims verification demand mechanically. |
| `core/skills/ship/SKILL.md` | Extended: §0 gains the third re-grounding check (`claims-check --diff`); §1's second-approver check now invokes the real script instead of inline logic. |
| `core/scripts/setup` | Extended: `--check` now also compares the committed `.claude/hook-guard` against this machine's spine template, independent of pin match — closes the hook-guard-maintenance question. |
| `core/scripts/ledger` | Extended: `ship_time_regrounding_claims` field. |
| `README.md` | Extended: hook-guard maintenance section, "Working with other engineers" section. |
| `docs/tradeoffs.md` | Extended: full Extension C section — cost, claims coarseness, approval-race/ship-time-regrounding linkage, identity trust model, the enforcement-tier sweep, laziest-defeat table, 2–4/8-engineer limits. |
| `docs/example/ext-c-two-engineer-demo/` | New. Real task-folder artifacts + README from the live demo below. |

## The two-engineer demo — real, through pushed registry state only

Alice: this machine, `~/spine`. Bob: `debian:bookworm-slim` container,
his own spine checkout at `/opt/spine` (a different absolute path — also
re-confirms 2.1's portability property in passing), his own git identity
configured fresh in the container. **At no point did Alice's and Bob's
checkouts share a filesystem** — every cross-engineer fact was learned
through `git push`/`git pull` against one bare `remote.git`. Full
transcripts are in this session; `docs/example/ext-c-two-engineer-demo/
README.md` is the durable artifact index. Summary:

- **(a) genuine claims conflict** — real write/read block
  (`src/lib/auth.ts`), Bob having learned of Alice's task only via the
  pushed registry; resolved by renegotiation (re-scoped to
  `src/routes/login.ts`/`src/lib/session.ts`), re-checked clear.
- **(b) mid-flight invalidation** — Alice ships a real change touching
  what Bob's task grounds on; `propagate` flags it; Bob's pull surfaces
  the flag; flag check refuses the phase advance; Bob acknowledges;
  advance succeeds; **ship-time re-grounding independently catches the
  same drift as `STALE`** afterward — acknowledgment doesn't silently
  clear the underlying grounding problem, recorded as a halt-tier
  deviation.
- **(c) Class 2, non-owner approval** — Bob approves Alice's plan from his
  own session, his own resolved git identity; `second-approver-check`
  (the real script, not hand-verified logic) gates the ship; both
  identities land in `ledger.json` (`engineer`, `second_approver`).
- **(d) per-engineer `/costs`** — real `ledger aggregate --by-engineer`
  output, Bob's row correctly carrying the `claims_conflicts: 2` and
  `avg_deviation_count: 1` his tasks actually generated, Alice's correctly
  at zero for both.
- **(e) pin skew** — deferred to *after* this phase's own delivery
  commit below, since a genuine skew test needs two real, different spine
  commits to exist; run and recorded at the end of this handoff.

## Solo-regression proof

Confirmed, not asserted: `~/bgr` (real project, zero open tasks) —
`claims-check`/`propagate` both no-op cleanly (`clear`/`nothing flagged`)
against an empty registry; `phase-gate` (via the now-real `hook-guard`
indirection) fires identically to pre-Extension-C behavior, exit 0, no
active task; `git status --short` in `~/bgr` clean after every test in
this phase — nothing left behind. This is the "zero behavioral change
except the fixed install mechanism" guarantee, checked against the real
project this build already migrated, not a synthetic fixture built to
pass.

## Self-red-team

Full detail lives in `docs/tradeoffs.md`'s new Extension C section (the
enforcement-tier sweep table and the laziest-defeat table) — not
duplicated here. Headline finding, stated once: **every gate this
extension added is enforced at the same tier the base system's own
pre-existing gates already were — a real script producing a real answer,
acted on by an agent following skill prose — except `hook-guard`, which is
the one mechanism a session cannot choose to bypass.** This was verified
empirically, not assumed: a direct `Write` to `work/<task-id>/state` with
an unacknowledged flag present passes all three real hooks with exit 0.
Given the build prompt's explicit "no new hooks" constraint, this is the
correct, disclosed shape of this build, not an oversight — but it needed
saying plainly rather than letting "real script, real exit code" imply
mechanical enforcement that doesn't exist at the skill-transition level.

Narrow-claims verification (the review's specific demand): resolved with
a real mechanism, not just documentation — `claims-check --diff`, tested
live against the actual defeat scenario (declared claims narrowed,
real diff still collides), tagged `[UNDECLARED]`, distinct from the
pre-existing generic `conformance` drift signal it used to be
indistinguishable from.

## Scenario (e) and delivery

Run after this phase's own commit lands (below), against the real
pre-commit sha and the real post-commit sha, in a fresh container —
recorded here once both exist:

```
<filled in after the commit below>
```

## What's left, named rather than hidden

- The briefing-aggregation and override-frequency counters named as gaps
  in `docs/tradeoffs.md`'s laziest-defeat table are real, disclosed,
  not-built v2 items — not silently deferred.
- No multi-repo (Extension B) scenario in the two-engineer demo — B's own
  contract mechanisms were already proven by the prior build; Extension C
  degrades to file/decision-only propagation when B is absent, confirmed
  by construction (`propagate`'s contract branch only ever matches
  against a non-empty `--contracts` list, never checks for
  `workspace.json`'s existence directly).
- `/task`'s step-0 pin-check halt and the flag-check's "withhold plan
  presentation" behavior were verified at the script-output level, not
  inside a live, uninterrupted `/task` session — the documented
  classifier wall (nested `claude` invocation from within this session's
  own Bash tool) makes that specific end-to-end drive unavailable to this
  build's own tooling, same limitation every phase of the prior extension
  build also hit and disclosed the same way.
