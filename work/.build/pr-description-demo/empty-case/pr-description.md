<!--
SYNTHETIC — constructed for spine/work/.build/pr-patch-phase-B-handoff.md
to prove the empty-case line renders correctly, not a real task. All four
unions are engineered to compute to empty: no deviations.md records, the
one adversary that ran kept zero verdicts, no protected-path glob is
touched, and conformance shows actual == predicted exactly.
-->

## synthetic-empty-case-demo — add a `formatCurrency` helper

**What & why:** Adds a small, pure `formatCurrency(cents: number): string`
helper and its unit tests. No existing code changes.

**Review this at the plan level:** New, isolated, pure function with no
callers yet — little to go wrong, per the plan's own risk section.

**How it was verified:**
- Floor: pass (typecheck, lint, test — 3/3 new unit tests PASS).
- Adversaries: Falsifier ran (adversarial inputs, stub-out probe) — 0
  verdicts kept, 0 dropped. Nothing to report.
- Plan accuracy: diff landed exactly where the plan said — precision 1.00,
  recall 1.00, f1 1.00.
- Gaps: none.

**Where to look (in priority order):** The diff landed exactly where the
approved plan predicted; no deviations, findings, or drift. Spot-check at
will.

**What surprised us:** Nothing — the plan held.

Record: `work/synthetic-empty-case-demo/` — plan, verify.md, deviations.md
(synthetic demo).
