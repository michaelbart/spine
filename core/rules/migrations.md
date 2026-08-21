---
paths:
  - "**/migrations/**"
  - "**/migration/**"
---

# Migration discipline

This rule loads whenever a file under a `migrations/`- or `migration/`-named
directory is read. It says nothing about a specific migration tool or
language — the capability that actually rehearses a migration is
`.spine/adapters/migrate-rehearse` (project-owned, stack-specific); this
rule is the always-applicable discipline around it.

**Expand/contract, not in-place.** A destructive schema change (dropping a
column, renaming a field, narrowing a type, removing an index another code
path relies on) never ships in the same change as the application code that
stops needing the old shape. Two tasks, in order:

1. **Expand.** Add the new shape alongside the old one. Application code
   migrates to read/write the new shape while the old shape still exists and
   still works for anyone not yet migrated.
2. **Contract.** Once nothing depends on the old shape — verified, not
   assumed — remove it in its own, later task.

A single task that both changes application code *and* destructively
changes the schema those changes depend on is a co-commit the deterministic
floor flags: it collapses expand/contract into one irreversible step, and it
is exactly the shape of change that breaks rollback.

**Every migration ships a rollback.** If you cannot articulate the rollback
for a schema change, you do not understand the change well enough to expand
it. The rollback is not a nice-to-have written after the fact — it's part of
what "the migration" means here.

**Backfills ship an invariant query.** A backfill that populates the new
shape from the old one ships alongside a query that checks the invariant the
backfill is supposed to establish (e.g. "every row in the new shape has a
corresponding row in the old shape with matching keys"). `migrate-rehearse`
runs this query on both sides of the migration — before and after — not just
after; a backfill that silently corrupts old-shape data on the way in is as
real a failure as one that produces a wrong new shape.

**`migrate-rehearse` is a Class 2 gate.** Against a seeded environment
(never production data — see the tradeoffs doc for what that concedes), it
runs, in order: migrate → verify invariants → rollback → re-migrate. All
four steps must succeed for the capability to pass. This is what makes
"ships a rollback" a checked claim instead of a comment.

**`migrate-rehearse` restores the stack, it doesn't reset it.** Rehearsing
a migration typically means bringing the local stack up to run it against
— but if the stack was already up before `migrate-rehearse` started, it
must still be up when `migrate-rehearse` exits, success or failure alike;
it stops the stack on the way out only if it started it. A capability
that unconditionally tears the stack down as its own cleanup step,
whatever state it found it in, breaks every capability that runs after it
in the same `floor` invocation and assumes the stack it just used is
still there (`ADAPTER-CONTRACT.md` §2.2, §3.7).

**On a schemaless store**, "migration" still means a change to a document
shape or an authorization/index rule that existing writers or readers depend
on — expand/contract and the invariant-query discipline apply the same way;
there is simply no separate migration-runner tool enforcing it, so
`migrate-rehearse` carries correspondingly more of the weight.
