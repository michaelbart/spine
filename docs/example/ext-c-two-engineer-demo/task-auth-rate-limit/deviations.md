# Deviations

- Tier: halt
- Status: resolved
- What: ship-time re-grounding (check-stale) found src/lib/session.ts drifted after task 20260811-billing-fee-calc shipped (flagged via propagate, acknowledged, then independently caught stale at re-grounding).
- Resolution: research regenerated against current HEAD before re-planning; counts toward the deviation circuit breaker per core/skills/ship/SKILL.md §0.
