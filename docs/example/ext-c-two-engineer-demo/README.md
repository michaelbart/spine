# Extension C worked demonstration — two engineers, pushed registry state only

Two real, independent checkouts of a bare remote — Alice on this machine,
Bob in a `debian:bookworm-slim` Docker container with his own spine
checkout at a different absolute path (`/opt/spine` vs `~/spine`) and his
own git identity (`Bob <bob@example.com>`, configured fresh in the
container, unrelated to this machine's global config). **No shared
filesystem between them at any point** — every cross-engineer fact below
was learned exclusively through `git push`/`git pull` against the bare
`remote.git`. Full command transcripts are in this build's own
conversation record; this file is the artifact index plus the load-bearing
output snippets.

## (a) Genuine claims conflict, learned only via the shared mainline

Alice opens `20260811-billing-fee-calc` (predicted touch:
`src/lib/billing.ts`, grounds on `src/lib/auth.ts`), pushes. Bob, in his
container, opens `20260811-auth-rate-limit` predicting `src/lib/auth.ts`
— a real write/read collision:

```
claims-check: BLOCKED — 1 conflict(s):
  - write/read with 20260811-billing-fee-calc (Alice <alice@example.com>):
    this task predicts touching 'src/lib/auth.ts', which
    20260811-billing-fee-calc's research grounds on
```

Resolved by renegotiation: Bob's task re-scoped to `src/routes/login.ts`,
grounded on `src/lib/session.ts` instead — `claims-check` re-run, clear.
Both artifacts kept (`task-billing-fee-calc/`, `task-auth-rate-limit/`).

## (b) Mid-flight invalidation, flag → acknowledge → re-grounding still catches it

Alice ships `20260811-billing-fee-calc` for real (a commit touching both
`src/lib/billing.ts` and `src/lib/session.ts` — the file Bob's task
grounds on), then runs `propagate`:

```
propagate: flagged 20260811-auth-rate-limit (1 entry, severity=advisory)
```

Bob's next `git pull --rebase` surfaces the flag purely through the
pushed registry. Flag check refuses the phase advance:

```
REFUSED — 1 unacknowledged flag(s)
  - file 'src/lib/session.ts' changed (task 20260811-billing-fee-calc shipped)
```

Bob acknowledges (`flags.json` edited, `registry-sync`'d) — advance
succeeds. **Then, independently, ship-time re-grounding still catches the
drift** — acknowledging a flag doesn't silently re-ground the research:

```
check-stale: STALE — 1 grounding item(s) changed since 08f49b8...:
  - src/lib/session.ts
```

Recorded as a halt-tier deviation (`task-auth-rate-limit/deviations.md`),
counted toward the circuit breaker (`ledger.json`'s `deviation_count: 1`).

## (c) Class 2, approved by the non-owner, shipped with both identities on record

Alice opens `20260811-auth-token-rotation` (Class 2), presents the plan.
Bob — reading it purely off the pushed `work/<task-id>/plan.md`-equivalent
state, no separate review tool — approves from his own session, his own
git identity resolved from his own `git config`, not typed:

```json
{"approver": "Bob <bob@example.com>", "at": "2026-08-11T13:15:00Z", "override": false, "override_reason": null}
```

Alice ships; the gate is a real script, not hand-verified logic:

```
second-approver-check: PROCEED — real second approver on record: Bob <bob@example.com>
```

`task-auth-token-rotation/ledger.json` carries both identities:
`"engineer": "Alice <alice@example.com>"`, `"second_approver": "Bob <bob@example.com>"`.

## (d) Per-engineer `/costs`

```
untracked-commit ratio: 1.00 (19/19 commits have no Spine-Task: trailer) — bypassed: 0
```

(Honest disclosure: this demo's own commits didn't replicate `/ship`'s
exact `Spine-Task:` trailer convention — a real `/ship` run would; this
number is a demo artifact, not evidence about the mechanism it's
measuring.)

```json
[
  {"engineer": "Alice <alice@example.com>", "task_count": 2, "class_escalation_count": 0,
   "bypass_count": 0, "tooling_gap_count": 0, "claims_conflicts": 0, "avg_deviation_count": 0},
  {"engineer": "Bob <bob@example.com>", "task_count": 1, "class_escalation_count": 0,
   "bypass_count": 0, "tooling_gap_count": 0, "claims_conflicts": 2, "avg_deviation_count": 1}
]
```

Bob's row correctly carries the two blocking `claims-check` hits from (a)
and the one deviation from (b) — Alice's doesn't. This is the instrument
build prompt §2.7 describes: visible, per-person, not a ranking.

## (e) Pin skew — see `../` build handoffs

Scenario (e) (engineer 2's core deliberately behind the pin) needed a real
committed spine core to be genuine — run separately, after this build's
own delivery commit, documented in
`work/.build/ext-c-phase-D-handoff.md` rather than duplicated here.

## What's *not* in this demo

No multi-repo/contract scenario — this demo project has no
`workspace.json`; Extension B's own worked examples (from the prior
build) already cover contract-touch/registry-staleness and aren't
re-derived here. No real `/verify` floor run — the demo project's
`.spine/capabilities.json` marks every capability `not-applicable`
(synthetic project, no real stack), so `floor` itself was never exercised
in this demo; it's the same script proven extensively in the original
build and untouched by Extension C.
