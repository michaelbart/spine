---
paths:
  - "contracts/**"
---

# Contract discipline

This rule loads whenever a file under a workspace's `contracts/<name>/`
directory is read. It generalizes `core/rules/migrations.md`'s
expand/contract discipline from one repo's schema to the boundary between
repos — spine's migration discipline at system scale.
Nothing here names a spec format (OpenAPI, protobuf, a hand-written
markdown table) — the registry and this rule are stack-blind; the spec
content itself is whatever the producer's stack actually needs.

**A breaking change to a contract spec never ships as one task.** Additive
changes (a new optional field, a new endpoint, a new enum value nothing
existing depended on the absence of) ship producer-then-consumer in a
single staged task, per `core/skills/ship/SKILL.md`'s declared ship order.
A breaking change — removing a field, narrowing a type, changing an
existing field's meaning, anything an existing consumer's current
implementation depends on that stops being true — decomposes into a
**milestone** of (at least) three member tasks, in order:

1. **Expand.** The producer ships *both* shapes side by side — the old one
   still fully functional, the new one available alongside it. Nothing
   about this task is breaking; it is purely additive from every existing
   consumer's point of view.
2. **Migrate.** Each consumer, in its own task (or one task per consumer if
   there's more than one), switches from the old shape to the new one.
   `contract-check` on that consumer must pass against the new shape before
   this task ships.
3. **Contract.** Once every consumer has migrated — verified via
   `contract-check` passing for all of them against the new shape, not
   assumed — the producer removes the old shape in its own, final task.

**The mechanical check, not just the discipline**: `core/scripts/
contract-touch` classifies every touched contract's spec diff as
`additive` (the diff contains only added lines) or `breaking` (the diff
contains any removed or modified line — a modified line is a remove+add
pair in a unified diff, so this catches in-place changes too, not just
outright deletions). `/verify`'s aggregation fails a task outright if any
touched contract classifies `breaking` unless that task's own `plan.md`
declares `expand` or `contract` in its `## Contract change` section (i.e.,
it is knowingly one leg of a decomposed milestone, per above) — this is
what makes "breaking contract changes are refused as a single task" a
checked claim instead of a written-down intention, and it is independent
of what the plan *claims* it's doing: a plan that calls itself additive
while its own diff removes or rewrites a line still fails, because the
classification comes from the diff, not the human's or the model's
say-so.

**Disclosed residual**: this mechanical check catches every
removal-or-modification-shaped break. It does **not** catch an
addition-shaped break — a newly *required* field is, line-for-line, a pure
addition to the spec, indistinguishable from a newly *optional* one by this
diff-only heuristic. Whether a new field is required or optional is
stack-specific spec semantics this stack-blind check does not parse. This
is a named, accepted gap (see `docs/tradeoffs.md`'s "Cross-repo work"
section) — a required-field addition must still be caught by the human
review the plan approval and `/verify`'s adversary review already provide,
not by this script.

**Registry staleness is a warning, not a gate.** `contract-touch` also
flags a contract whose `producer_paths` glob currently matches zero files
in the producer repo, when the registry's own `producer_paths_match_count`
recorded a nonzero count at registration — a strong signal the producer
refactored those paths away without updating the registry entry, which
means blast-radius detection for this contract may now be silently wrong.
Update the registry entry's `producer_paths` (and
its match count) as part of whatever task caused the drift; don't let a
stale registry entry ride along unexamined just because nothing failed.
